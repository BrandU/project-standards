# General Conventions #
Formatting, naming, structuring etc.

__Aim__:
* well documented,
* consistent,
* readability / KISS,
* re-usability / DRY,
* following the best practices & standards.

### General rules ###
* no unused code,
* consistent formatting,
  * every section & object definition should be separated by the new line,
  * definition properties always ordered for each object definition type (parameter, response object etc.)

### Do not repeat yourself ###
In case some objects are reused through the API, always create them within components and reference 
them where needed using __$ref__.

**Sample**

```yaml
...

'/api/v1/identities/parties/{partyId}':
  get:
    parameters:
      - $ref: '#/components/parameters/partyId'

...

  parameters:
    partyId:
      name: partyId
      in: path
      description: Unique identifier of the party. Also known as customer number.
      required: true
      schema:
        type: string
```

### Enum values
Enum values are following the Java programming language conventions. 
Uppercase value names, connecting words with "_". Hyphen / "-" is also allowed when the standard wording contains it,

e.g.:
 * CO-DEBTOR, 
 * SELF-EMPLOYED_ENTREPRENEUR.

**Sample**:

```yaml
enum:
  - SELF-EMPLOYED
  - SME
  - PRIVATE_EMPLOYED
```

### Properties, attributes, parameters
#### Naming
Convention: camelCase

**Sample**:
```yaml
properties:
   partyId:
     type: string
     description: Unique identifier of the party.
```

### Response objects
#### Naming
Response objects (root or nested) naming follows Java language conventions.
Convention: PascalCase
 
**Sample**:

```yaml
RiskProfile:
   type: object
   description: List of variables related to party in risk domain.
   properties:
     partyId:
       type: string
       description: Unique identifier of the party.
```

### OperationId ###
OperationId should follow Java method naming convention.
{verb}{Entity}{API_version}

**Sample**

```yaml
'/api/v1/products/contracts':
    get:
      operationId: getContractsV1
```

### Paging attributes & parameters ###
This section discusses the standard approach for paging in the APIs.

#### Paging query parameters ####
Standard set of query parameters required on the "pageable" resources.

```yaml
    page:
      name: page
      in: query
      style: form
      description: Page number query parameter.
      schema:
        title: page
        type: integer
        default: 1
        minimum: 1
        maximum: 32768
        multipleOf: 1

    size:
      name: size
      in: query
      style: form
      description: Page size query parameter.
      schema:
        title: size
        type: integer
        default: 10
        minimum: 1
        maximum: 100
        multipleOf: 1

    sort:
      name: sort
      in: query
      style: form
      description: Sort by string query parameter.
      schema:
        title: sort
        type: string
        default: ""
```


#### Page response object explained ####
The page abstraction contains these properties:

```yaml
    Page:
      title: Page
      type: object
      required:
        - pageType
      properties:
        pageType:
          type: string
          description: 
            Attribute used as discriminator, no business value, just for the polymorphism & inheritance purposes.
            So that the API supports multiple subtypes of Page differing in the content type only.
        empty:
          type: boolean
          description: Flag whether the page content is empty
        first:
          type: boolean
          description: Flag whether the page returned is the first one
        last:
          type: boolean
          description: Flag whether the page returned is the last one
        number:
          type: integer
          format: int32
          description: Page number, starting from 1
        numberOfElements:
          type: integer
          format: int32
          description: Number of elements returned on the current page.
        size:
          type: integer
          format: int32
          description: Page size / number of elements requested per page
        totalElements:
          type: integer
          format: int64
          description: Total number of elements matching the input query
        totalPages:
          type: integer
          format: int32
          description: Total number of pages matching the input query
      discriminator:
        propertyName: pageType
        # all the implementations of the Page / subtypes / children, used for better abstraction.
        mapping:
          pageOfParties:    '#/components/schemas/PageOfParties'
          pageOfAccounts:   '#/components/schemas/PageOfAccounts'
          pageOfContracts:  '#/components/schemas/PageOfContracts'
```

### Standard error responses ###
Always define standard responses under schema.components.responses & re-use them in all resources where applicable.
The error bodies should be defined under schema.components.schemas.

Following these guidelines will result into clean code on a client's / server's end & clean display in Swagger UI:



#### Responses & response schemas definitions ####
Responses are defined like this, again re-using common error definitions.

```yaml
  schemas:
    ...
    #Parent error definition 
    ErrorResponse:
    type: object
    required:
      - errorType
    properties:
      status:
        type: integer
        description: HTTP Status
      code:
        type: string
        description: Business code of error.
      message:
        type: string
        description: Error message
      detail:
        type: string
        description: Detail information about error.
      errorType:
        type: string
        description: Discrimator attribute to distinguish between various error types.
    discriminator:
      propertyName: errorType
      mapping:
        badRequestError   : '#/components/schemas/BadRequestError'
        unauthorizedError : '#/components/schemas/UnauthorizedError'
    ...
    # Concrete error implementation extending general error
    BadRequestError:
      allOf:
        - $ref: '#/components/schemas/ErrorResponse'
      properties:
        errorType:
          type: string
          default: 'badRequestError'
          example: 'badRequestError'
        status:
          type: integer
          description: HTTP Status
          example: 400
          default: 400
    ...
    
  responses:

    BadRequestResponse:
      description: Bad request error response.
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/BadRequestError'

```

#### Response definition usages ####
Reference the standard set of error responses like in the example below.

```yaml
  paths:
  '/api/v1/identities/parties':
    get:
      ...
      responses:
        '400':
          $ref: '#/components/responses/BadRequestResponse'
        '401':
          $ref: '#/components/responses/UnauthorizedResponse'
        '403':
          $ref: '#/components/responses/ForbiddenResponse'
        '404':
          $ref: '#/components/responses/NotFoundResponse'
        default:
          $ref: '#/components/responses/UnexpectedErrorResponse'      
```

#### Use-cases for concrete errors returned ####
* **400 / Bad request error**:
    * For queryable GET resources: expected query parameter is in a form of integer number, yet the caller sets an alpha string,
* **401 / Unauthorized error**:
    * When the concrete API resource is secured & requires authorization and not present in the request,
* **403 / Forbidden error**:
    * The authorization is valid, however the resource requested is not accessible for this user / role,
* **404 / Not found error**:
    * For GET resource with path parameter of identifier, if the non-existing identifier value is set,
* **50X / Unexpected error**:
    * These are fatal / unrecoverable server errors which are never to be thrown intentionally,
    * E.g., DB connectivity problem, unhandled exception on server side etc.


### Security - OAuth2 ###
The definition of the security goes to the end of the specification file under schema.components.securitySchemas.

The definition looks like this:

```yaml
securitySchemes:

    keycloak: # chosen name of the security schema
      type: oauth2
      description: This API uses OAuth2 with the implicit grant flow integrating the SCB Keycloak service.
      flows:
        implicit:
          authorizationUrl: https://api.example.com/oauth2/authorize
          scopes: # scopes are used under actual resources
            read_risk: 'read risk data'
            read_identities: 'read identities data'
            read_products: 'read products data'
            read_applications: 'read applications/decisions data'
```

#### Usage ####
If we want to lock resources / make them accessible for only specific scope:

```yaml
paths:
  '/api/v1/identities/parties':
    get:
      tags:
        ...
      parameters:
       ...
      responses:
        ...
      security: # this resource is only available in the given scope
        - keycloak:
            - read_identities 
```
