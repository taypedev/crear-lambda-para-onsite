## Paso 1: Define la ruta (path) de tu endpoint

Edita `lib/lambda/common/paths-api.ts` y agrega una constante o una clave nueva dentro del grupo correspondiente. Por ejemplo, para un módulo "orders":

```ts
export const API_ORDERS_PATHS = {
  create: "orders/create",
  getById: "orders/get-by-id",
};
```

Consejos:
- Mantén nombres consistentes con los existentes (kebab-case en la URL).
- Agrupa por dominio cuando tenga sentido (como `API_PROFILE_PATHS`, `API_PAYMENT_PATHS`).

## Paso 2: Crea (o reutiliza) el rol IAM de la Lambda

1) Si un rol existente no cubre tu caso, agrega uno nuevo en `lib/lambda/roles/lambda-roles.ts`:

```ts
{
  id: "ordersCreateLambdaRole",
  assumedByService: "lambda.amazonaws.com",
  managedPolicies: ["service-role/AWSLambdaBasicExecutionRole"],
  policyStatements: [CloudWatchLogPermissions],
  additionalPolicies: [
    "SSMReadAccessPolicy",
    "S3ReadAccessPolicy", // Opcional según tu caso
  ],
},
```

2) Si necesitas permisos adicionales (por ejemplo, otro bucket, DynamoDB, etc.), defínelos en `lib/lambda/roles/lambda-policies.ts` dentro de `createPolicies` y referencia la clave desde `additionalPolicies`:

```ts
CustomOrdersBucketPolicy: new PolicyStatement({
  actions: ["s3:GetObject", "s3:PutObject"],
  effect: Effect.ALLOW,
  resources: [
    "arn:aws:s3:::mi-bucket-orders",
    "arn:aws:s3:::mi-bucket-orders/*",
  ],
}),
```

Nota: `createPolicies` recibe `region` y `account`, útil para construir ARNs de SSM, etc.

## Paso 3: Crea la “factory” de tu Lambda

Añade un archivo en `lib/lambda/functions/<modulo>/<nombre-funcion>.ts` que exporte una función que devuelva un `NodejsFunction`. Ejemplo mínimo (similar a los existentes):

```ts
import { Role } from "aws-cdk-lib/aws-iam";
import { Code, Runtime } from "aws-cdk-lib/aws-lambda";
import { NodejsFunction } from "aws-cdk-lib/aws-lambda-nodejs";
import { Duration } from "aws-cdk-lib/core";
import { Construct } from "constructs";
import { LambdaLayerStack } from "../../lambda-layer-stack";

interface LambdaProps {
  lambdaRole: Role;
  layerStack: LambdaLayerStack;
  scope: Construct;
}

const PATH_ORDERS_FUNCTIONS = "lib/lambda/src/functions/orders";
const PATH_CREATE_ORDER = "create-order";
const CREATE_ORDER_FUNCTION_NAME = "createOrderFunction";

export const createOrderLambda = (props: LambdaProps) => {
  return new NodejsFunction(props.scope, CREATE_ORDER_FUNCTION_NAME, {
    functionName: CREATE_ORDER_FUNCTION_NAME,
    memorySize: 1024,
    timeout: Duration.seconds(30),
    runtime: Runtime.NODEJS_20_X,
    code: Code.fromAsset(`${PATH_ORDERS_FUNCTIONS}/${PATH_CREATE_ORDER}`),
    handler: "handler.handler",
    role: props.lambdaRole,
    layers: [
      props.layerStack.utilsLayer,
      // props.layerStack.pgLayer, // Activa si necesitas PostgreSQL
      // props.layerStack.s3Layer, // Activa si necesitas S3 helpers
    ],
    bundling: {
      minify: true,
      externalModules: ["axios"],
    },
  });
};
```

Capas disponibles (según `lambda-layer-stack.ts`): `utilsLayer`, `pgLayer`, `s3Layer`, `paymentLayer`, `paymentLayer2`.

## Paso 4: Crea el código del handler

Crea la carpeta de código fuente en `lib/lambda/src/functions/<modulo>/<carpeta>/` con un `handler.js` (o `.ts`) que exporte `handler`. Ejemplo base:

```js
// lib/lambda/src/functions/orders/create-order/handler.js
const moment = require("moment-timezone");

exports.handler = async (event) => {
  try {
    const body = event.body ? JSON.parse(event.body) : {};
    // TODO: lógica de negocio (validaciones, llamadas a DB/API, etc.)

    return buildResponse(200, true, { message: "Orden creada", input: body });
  } catch (err) {
    console.error("create-order error", err);
    return buildResponse(500, false, { message: "Error interno", error: err.message });
  }
};

function buildResponse(statusCode, success, data) {
  const timezone = "America/Lima";
  const timestampUTC = new Date().toISOString();
  const timestampLocal = moment().tz(timezone).format("YYYY-MM-DD HH:mm:ssZ");

  return {
    statusCode,
    body: JSON.stringify({ success, statusCode, timestampUTC, timestampLocal, data }),
    headers: { "Content-Type": "application/json" },
  };
}
```

Si accedes a SSM o S3, recuerda que tu rol debe tener las políticas necesarias (ver Paso 2).

## Paso 5: Expón el endpoint en API Gateway

1) Crea (o reutiliza) un archivo en `lib/lambda/api-methods/<modulo>-api.ts` y agrega una función que use `addApiMethodsWithLambda` con tu ruta y método(s):

```ts
import { AuthorizationType, CognitoUserPoolsAuthorizer, RestApi } from "aws-cdk-lib/aws-apigateway";
import { Function } from "aws-cdk-lib/aws-lambda";
import { addApiMethodsWithLambda } from "../helpers/add-api-methods";
import { API_ORDERS_PATHS } from "../common/paths-api";

interface ApiConfigProps {
  restApi: RestApi;
  lambdaFunction: Function;
  authorizer: CognitoUserPoolsAuthorizer;
}

export const configureCreateOrderApi = ({ restApi, lambdaFunction, authorizer }: ApiConfigProps) => {
  addApiMethodsWithLambda({
    restApi,
    lambdaFunction,
    authorizer,
    resourcePath: API_ORDERS_PATHS.create,
    methods: [
      { method: "POST", authorizationType: AuthorizationType.COGNITO, useAuthorizer: true },
    ],
  });
};
```

Puntos clave del helper `addApiMethodsWithLambda`:
- `resourcePath` acepta rutas con `/` y crea recursos anidados.
- `AuthorizationType`: usa `COGNITO` + `useAuthorizer: true` para proteger el endpoint con Cognito; usa `NONE` para público.
- `queryParameters: ["param1", "param2"]` permite declarar parámetros de consulta opcionales.

## Paso 6: Cablea todo en `LambdaFunctionStack`

Edita `lib/lambda/lambda-functions-stack.ts`:

1) Importa tu factory y tu configurador de API.
2) Dentro del constructor, instancia la Lambda pasando el rol correcto desde `props.lambdaRoles["<RoleId>"]`.
3) Llama a tu función `configure*Api` para agregar el recurso y método(s).

Ejemplo:

```ts
import { createOrderLambda } from "./functions/orders/create-order";
import { configureCreateOrderApi } from "./api-methods/orders-api";

// ... dentro del constructor
const createOrderFn = createOrderLambda({
  lambdaRole: props.lambdaRoles["ordersCreateLambdaRole"],
  layerStack: props.layerStack,
  scope: this,
});

configureCreateOrderApi({
  restApi: props.restApi,
  lambdaFunction: createOrderFn,
  authorizer: props.authorizer,
});
```

## Paso 7: Despliegue

Compila, sintetiza y despliega. En PowerShell:

```powershell
npm run build
npx cdk synth
npx cdk deploy
```
