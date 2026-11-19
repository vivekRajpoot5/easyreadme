# Easy Housing Solution - Complete Project Flow

## 1. What this project does

Easy Housing Solution is a property marketplace with four kinds of people:

| Person | What they can do |
|---|---|
| Visitor | Search and open public property listings |
| Buyer | Compare properties, save properties, view owner contact details |
| Seller | Add or edit properties, upload photos, and view listing status |
| Admin | Verify, search, and deactivate properties |

The project is divided into two main parts:

1. **Frontend**: the Angular website that the person uses in a browser.
2. **Backend**: a set of .NET API services that process requests and store data.

An API Gateway is the common front door for the backend. The browser does not need to know the address of every separate backend service.

## 2. Simple big picture

```text
Browser
  |
  | Angular page, button click, form submission
  v
Angular Components
  |
  | Angular Services
  v
HTTP Interceptors
  |
  | http://localhost:5200/api/...
  v
API Gateway on port 5200
  |
  +--> Identity Service on port 5201
  +--> User Service on port 5202
  +--> Property Service on port 5203
  +--> Cart Service on port 5204

Each backend service talks to its own SQL Server database.
Some services call another service internally.
```

Think of the system like this:

- **Angular components** are the screens and visible buttons.
- **Angular services** are the messengers that send browser requests.
- **HTTP interceptors** prepare requests and handle common errors.
- **The API Gateway** decides which backend service should receive a request.
- **Backend application services** apply the business rules.
- **Backend repositories** read and write database tables.
- **Backend controllers** receive HTTP requests and send HTTP responses.

## 3. Project structure

### 3.1 Frontend structure

```text
frontend/
  src/
    main.ts                     Starts the Angular application
    server.ts                   Runs the Angular SSR Express server
    app/
      app.ts                    Root application shell
      app.routes.ts             Maps browser URLs to pages
      app.config.ts             Registers router and HTTP interceptors
      core/
        api.config.ts           Shared API base address
        guards/                 Protects pages by login role
        interceptors/          Adds tokens and handles errors
        models/                 TypeScript shapes for API data
        services/               Calls backend endpoints
        utils/                  Small reusable helpers
      features/
        auth/                   Login and registration pages
        buyer/                  Search, details, cart, and compare pages
        seller/                 Seller listing dashboard
        admin/                  Admin moderation dashboard
        shared/                 Navbar and error page
```

### 3.2 Backend structure

```text
backend/
  services/
    ApiGateway/                 Common public entry point
    IdentityService/            Accounts and login tokens
    UserService/                Profiles, states, cities, and contact data
    PropertyService/            Property listings, images, search, moderation
    CartService/                Buyer saved-property records
```

The backend is a microservice-style project. It is not one very large API. Each service has a similar internal structure:

```text
Controller
  receives the HTTP request
  |
Application Service
  applies business rules
  |
Repository
  reads/writes database records
  |
Database
```

## 4. How the application starts

### 4.1 Frontend startup

The browser loads `src/main.ts`.

```text
src/main.ts
  -> bootstrapApplication(App, appConfig)
  -> appConfig provides Angular Router and HttpClient interceptors
  -> App component renders Navbar + RouterOutlet + Footer
  -> Angular Router opens the URL that the browser requested
```

The root component is `src/app/app.ts`. It contains:

- The top navigation bar.
- `<router-outlet />`, which shows the current page.
- The footer.

The Angular development server normally runs on `http://localhost:4200`.

The project also supports server-side rendering. `angular.json` points to `src/main.ts` for the browser and `src/server.ts` for the SSR Express server. The server currently renders all routes in client mode.

### 4.2 Backend startup

Run the backend services on the following ports:

| Component | Port |
|---|---:|
| API Gateway | 5200 |
| Identity Service | 5201 |
| User Service | 5202 |
| Property Service | 5203 |
| Cart Service | 5204 |
| Angular frontend | 4200 |

Each backend service:

1. Creates a database context.
2. Configures controllers and Swagger.
3. Configures JWT authentication when protected endpoints are present.
4. Calls `Database.EnsureCreatedAsync()`.
5. Starts listening on its configured port.

The API Gateway uses `backend/services/ApiGateway/appsettings.json` to learn the addresses of the other services.

## 5. How a normal request travels

Example: a buyer searches for homes.

```text
1. User opens /search.
2. Angular Router loads PropertySearchComponent.
3. The component calls PropertyService.searchProperties(...).
4. PropertyService creates:
   GET http://localhost:5200/api/properties/search
5. The auth interceptor sees that this is an API request.
6. If a token exists, the interceptor adds an Authorization header.
7. The API Gateway reads the first path segment: "properties".
8. RouteServiceResolver sends the request to Property Service at port 5203.
9. PropertiesController receives GET /api/properties/search.
10. PropertyApplicationService asks PropertyRepository to search the database.
11. PropertyRepository returns matching property records.
12. The application converts those records into API response objects.
13. PropertyController sends the result to the gateway.
14. The gateway returns the JSON result to Angular.
15. PropertySearchComponent stores the result in a signal.
16. Angular redraws the page and displays the property cards.
```

The gateway does not run the search itself. It is a forwarding and routing layer.

### 5.1 What interceptors do

There are two HTTP interceptors.

**Auth interceptor**

It is used for requests sent to `http://localhost:5200/api`.

When the user is logged in, it reads `ehs_token` from browser `localStorage` and adds the token to the request header. This allows backend services to recognize the user's role and profile ID.

The interceptor does not run for:

- Images or static resources.
- Other websites.
- Server-side requests that do not use a browser.

**Error interceptor**

It watches all Angular HTTP responses. It sends especially serious failures to the common error page:

- Status `0`: the browser could not reach the backend.
- Status `404`: the requested resource was not found.
- Status `500` or higher: the server encountered a failure.

Property search is handled locally in its component instead of always navigating to the error page.

## 6. Frontend routing

The route map is in `src/app/app.routes.ts`.

| URL | Page | Who can enter |
|---|---|---|
| `/search` | Property search | Everyone |
| `/property/:id` | Property details | Everyone |
| `/login` | Login | Everyone |
| `/register` | Registration | Everyone |
| `/compare` | Compare properties | Buyer only |
| `/cart` | Saved homes | Buyer only |
| `/seller` | Seller dashboard | Seller only |
| `/admin` | Admin dashboard | Admin only |
| `/error` | Common error page | Everyone |

The role guard in `core/guards/role.guard.ts` checks the user stored in `AuthService`.

```text
No logged-in user
  -> redirect to /login

Logged in as the wrong role
  -> redirect to /error with status 403

Logged in as the correct role
  -> allow the page to open
```

The frontend guard improves navigation and user experience, but backend authorization is more important. Protected backend endpoints check the JWT role again. Someone could bypass the frontend, but they would still be rejected by the API if they do not have the required role.

## 7. Login and session flow

### 7.1 Login request

When the user submits the login form:

```text
LoginComponent
  -> AuthService.login()
  -> POST /api/identity/login
  -> API Gateway
  -> Identity Service
```

Identity Service:

1. Finds the account by username.
2. Compares the entered password with the stored password hash.
3. Returns no login session if the password is wrong.
4. Creates an eight-hour JWT when credentials are valid.

The JWT contains:

- Username.
- User role: `Buyer`, `Seller`, or `Admin`.
- Profile ID used to identify the person's profile.

### 7.2 Where login data is stored

After a successful login, `AuthService` stores:

```text
ehs_user  -> username, role, profile ID, and token
ehs_token -> the JWT used for protected API requests
```

Both are stored in browser `localStorage`.

`AuthService.currentUser` is an Angular signal. When its value changes:

- The navbar adds the username.
- The navbar shows only the links for that role.
- Role-protected pages can immediately read the role.

On logout:

1. `ehs_user` is removed.
2. `ehs_token` is removed.
3. `currentUser` becomes `null`.
4. Angular navigates to `/login`.

## 8. Registration flow

The registration page asks for:

- Username and password.
- First and last name.
- Date of birth.
- Phone number.
- Email address.
- Address and location for a seller.

The page loads states from `User Service` and loads cities after the user chooses a state.

```text
RegisterComponent
  -> LocationService.getStates()
  -> GET /api/locations/states
  -> gateway
  -> User Service
```

Choosing a state causes another request:

```text
GET /api/locations/states/{stateId}/cities
```

When the form is submitted:

```text
RegisterComponent
  -> AuthService.registerBuyer(...)
     or AuthService.registerSeller(...)
  -> POST /api/identity/register
  -> API Gateway
  -> Identity Service
```

Identity Service then performs a two-service operation:

```text
Identity Service
  -> checks that the username is not already used
  -> sends profile data to User Service
  -> receives a profile ID
  -> creates and hashes the identity account
  -> saves the account and profile ID together
```

The password is stored as a hash, not as plain text.

If creating the identity account fails after the profile was created, Identity Service asks User Service to delete the profile. This compensation step prevents an account profile from remaining while login credentials are missing.

After successful registration, the person is sent to `/login`.

## 9. Property search flow

The `/search` page is the public property catalogue.

The search component can send:

- Region.
- Property type.
- Listing option: sell or rent.
- Sort option.

The request is:

```text
GET /api/properties/search?region=...&type=...&sortBy=...&option=...
```

Flow:

```text
PropertySearchComponent
  -> PropertyService.searchProperties()
  -> API Gateway
  -> Property Service
  -> PropertyApplicationService.SearchAsync()
  -> PropertyRepository
  -> SQL database
  -> public verified property records
  -> Angular property cards
```

Public search shows only active, verified properties. Pending and deactivated properties are not returned.

## 10. Property details flow

Opening `/property/123` creates this flow:

```text
Angular Router reads id=123
  -> PropertyDetailsComponent
  -> PropertyService.getPropertyDetails(123)
  -> GET /api/properties/123
  -> Property Service
  -> finds only an active public property
  -> returns property details or 404
  -> component updates its property signal
  -> page displays the result
```

Owner contact information is separate. It is not included automatically in the public property response. The buyer must click **Contact the owner**.

The owner contact flow is:

```text
Buyer clicks Contact the owner
  -> frontend verifies that current user is a Buyer
  -> GET /api/properties/{propertyId}/contact
  -> Property Service validates Buyer JWT role
  -> Property Service finds the seller profile ID
  -> Property Service calls User Service
  -> User Service returns the owner name, email, and phone
  -> frontend displays contact details
```

This internal User Service call requires the shared internal API key.

## 11. Save to cart flow

This action belongs to a Buyer.

```text
Buyer clicks Save
  -> frontend checks that there is a logged-in Buyer
  -> PropertyService.addToCart(propertyId)
  -> POST /api/cart
  -> request body: { "propertyId": 123 }
  -> API Gateway
  -> Cart Service
  -> CartController reads buyer profileId from JWT
  -> CartApplicationService validates the property
  -> Cart Service calls Property Service
  -> Property Service confirms that the property is available
  -> Cart repository saves the buyer/property pair
  -> success or duplicate/conflict response
  -> Angular displays a success or error message
```

The cart item stores references, not copied property details:

```text
CartId
BuyerId
PropertyId
```

When `/cart` is opened:

```text
CartComponent
  -> GET /api/cart
  -> Cart Service reads saved rows for the JWT buyer
  -> for each property ID:
       ask Property Service for current property details
  -> cart response includes property details where available
  -> Angular displays saved homes
```

If a property was removed or deactivated, its cart entry can remain, but the property response is nullable and the page explains that the home is unavailable.

Removing an item is:

```text
DELETE /api/cart/{cartId}
  -> Cart Service verifies cart item ownership
  -> deletes it only if it belongs to the JWT buyer
```

The frontend buyer ID is not accepted as the authority. Cart Service extracts the buyer profile ID from the signed JWT.

## 12. Compare properties flow

The search page allows a Buyer to select exactly two properties.

```text
User selects property 1
  -> compareList becomes [id1]

User selects property 2
  -> compareList becomes [id1, id2]

User clicks Compare homes
  -> Angular navigates to /compare?id1=...&id2=...
```

The compare component reads both IDs from the URL and sends:

```text
GET /api/properties/compare?id1=123&id2=456
```

Property Service:

1. Requires the Buyer role.
2. Rejects missing, negative, or equal IDs.
3. Loads exactly two public properties.
4. Returns an error if either property is unavailable.

Angular shows the two property cards side by side.

## 13. Seller listing flow

The `/seller` page is protected by the role guard and also protected by backend Seller authorization.

### 13.1 Loading the seller page

When it opens:

```text
SellerDashboardComponent
  -> load states
  -> GET /api/locations/states
  -> load verified owned listings
  -> GET /api/properties/mine?status=Verified
  -> load all owned listings
  -> GET /api/properties/mine
```

The component uses the seller profile ID stored in the login session.

### 13.2 Creating a listing

The seller fills in:

- Property name.
- Property type.
- Sell or rent.
- Address.
- State and city.
- Landmark.
- Price.
- Initial deposit.
- Description.

Submitting creates:

```text
POST /api/properties
```

Property Service:

1. Reads the seller profile ID from the JWT.
2. Validates required fields.
3. Calls User Service to validate the state/city pair.
4. Converts the pair into a display region such as `City, State`.
5. Creates the listing with status `Pending`.
6. Sets `IsActive` to `false`.
7. Saves the listing in the property database.

The property does not appear in public search while it is pending.

### 13.3 Editing a listing

The seller clicks edit. Angular puts existing listing values into the form.

Submitting edit sends:

```text
PUT /api/properties/{propertyId}
```

Property Service:

1. Confirms the property belongs to the JWT seller.
2. Updates the listing.
3. Returns the listing to `Pending`.
4. Deactivates it until an Admin reviews it again.

### 13.4 Uploading images

The seller selects up to six files at a time.

```text
SellerDashboardComponent
  -> PropertyService.uploadImages(propertyId, files)
  -> POST /api/properties/{propertyId}/images
  -> multipart/form-data
```

Property Service validates:

- The property belongs to the seller.
- There are no more than six images per listing.
- Each file is JPEG, PNG, or WebP.
- Each file is no larger than 5 MB.
- The file is not empty.

The backend stores image binary data and content type in the property database.

## 14. Admin moderation flow

The `/admin` page is protected for the Admin role.

### 14.1 Pending verification queue

Opening the page sends:

```text
GET /api/properties/admin/pending
```

The component displays properties with status `Pending`.

### 14.2 Verifying a property

The Admin clicks **Verify**:

```text
PUT /api/properties/admin/{propertyId}/verify
  -> property status becomes Verified
  -> isActive becomes true
  -> old deactivation reason is removed
  -> property appears in public search
```

The component refreshes both the pending queue and any current search.

### 14.3 Deactivating a property

The Admin chooses a property and enters a reason:

```text
PUT /api/properties/admin/{propertyId}/deactivate
  -> body: { "reason": "..." }
  -> property status becomes Deactivated
  -> isActive becomes false
  -> reason is stored
  -> property disappears from public search
```

### 14.4 Searching properties as Admin

An Admin can search by region or seller profile ID:

```text
GET /api/properties/admin/region/{region}
GET /api/properties/admin/owner/{sellerId}
```

These endpoints are protected and return listings that can include statuses outside the public catalogue.

## 15. Gateway routing

The gateway uses the first part of the API URL.

| Browser path | Gateway route | Destination |
|---|---|---|
| `/api/identity/...` | Identity route | Identity Service |
| `/api/locations/...` | User route | User Service |
| `/api/internal/users/...` | Internal route | User Service |
| `/api/properties/...` | Property route | Property Service |
| `/api/cart/...` | Cart route | Cart Service |

`RouteServiceResolver` decides the destination.

The gateway:

- Copies the method, body, query string, request headers, and content headers.
- Sends the request to the downstream service.
- Copies the downstream status code, response headers, and response body back to the browser.
- Applies CORS for `http://localhost:4200`.

The gateway does not replace backend role checks.

## 16. Backend controller, application, and repository roles

### Controller

A controller receives an HTTP route and parameters.

Examples:

- `PropertiesController`
- `IdentityController`
- `CartController`
- `UsersController`

Controllers should be thin. They:

- Read route, query, body, and JWT claims.
- Call an application service.
- Convert the result into an HTTP response.

### Application service

An application service contains use-case rules.

Examples:

- Verify a password.
- Validate a property submission.
- Prevent duplicate cart entries.
- Confirm property ownership.
- Move a property to pending or verified status.

### Repository

A repository performs database operations.

Examples:

- Find a property.
- Add a property.
- Load a buyer's cart entries.
- Find an account by username.
- Save changes to SQL Server.

This separation makes each responsibility easier to understand:

```text
controller = HTTP connection
application = business rules
repository = database connection
```

## 17. JWT and role protection

After login, requests to protected endpoints carry a bearer token.

A service can then read claims from the token:

- `Name`: username.
- `Role`: `Buyer`, `Seller`, or `Admin`.
- `profileId`: database profile identifier.

Example backend authorization:

- `Authorize(Roles = "Buyer")`
- `Authorize(Roles = "Seller")`
- `Authorize(Roles = "Admin")`

JWT validation checks:

- Correct issuer.
- Correct audience.
- Correct signing key.
- Token lifetime.

The frontend route guard checks the session copy in the browser, but the backend JWT is the real security boundary.

## 18. Ownership and security rules

The project applies several ownership rules:

| Operation | Backend protection |
|---|---|
| Seller edits property | Property must belong to JWT seller profile |
| Seller uploads images | Property must belong to JWT seller profile |
| Buyer views cart | Cart rows use JWT buyer profile |
| Buyer removes cart item | Cart item must belong to JWT buyer profile |
| Buyer saves property | JWT must have Buyer role |
| Admin verifies property | JWT must have Admin role |
| Seller creates property | JWT must have Seller role |

Password and token security details are handled in backend configuration and code, not in the Angular display components.

Internal service-to-service routes in User Service require `X-Internal-Api-Key`.

## 19. Data stored by the system

The services use separate databases:

| Service | Main data |
|---|---|
| Identity Service | Usernames, roles, password hashes, profile IDs |
| User Service | User profiles, states, cities, contact details |
| Property Service | Property details, status, owner ID, image data |
| Cart Service | Buyer ID and property ID pairs |

### Property lifecycle

```text
Seller creates property
  -> Pending
  -> inactive

Admin verifies property
  -> Verified
  -> active
  -> visible in public search

Seller edits verified property
  -> Pending
  -> inactive
  -> waits for verification again

Admin deactivates property
  -> Deactivated
  -> inactive
  -> hidden from public search
  -> reason is recorded
```

## 20. Models and signals

### Models

`src/app/core/models/ehs.models.ts` contains TypeScript interfaces used by Angular. They describe the shape of objects received or sent through APIs:

- `UserSession`
- `Property`
- `PropertySubmission`
- `OwnerContact`
- `CartItem`

Interfaces help Angular and TypeScript catch mistakes while the code is being written.

### Signals

Angular components use signals for their changing UI state.

Common signals include:

- `isLoading`
- `isSubmitting`
- `errorMessage`
- `properties`
- `cartItems`
- `currentUser`

Example:

```text
isLoading starts as true
  -> HTTP request begins
  -> response arrives
  -> isLoading becomes false
  -> Angular renders the result
```

The browser does not continuously poll. A component starts a request and updates the signal when the request finishes.

## 21. Image handling

Each property may have up to six images.

When Angular receives an image:

1. It looks at the first image in `property.images`.
2. `propertyImageUrl()` checks if the image is already a URL or data URL.
3. It converts raw base64 image data into a browser image data URL.
4. If no valid image exists, Angular uses `/property-placeholder.svg`.

This logic is in `src/app/core/utils/property-image.ts`.

## 22. Error handling

Errors can appear at different levels.

### Angular form validation

Reactive forms prevent invalid values from being submitted.

Examples:

- Required field.
- Password length.
- 10-digit phone number.
- Future date of birth.
- Positive property price.

Frontend validation is only for user guidance. The backend validates again.

### HTTP component handling

Each component may display a page-specific message, for example:

- Search unavailable.
- Saved home could not be loaded.
- Property could not be verified.
- Photos could not be uploaded.

### Common error interceptor

Network errors and serious server statuses can navigate to `/error`.

The error page receives a message and status through Angular router state or query parameters.

### Backend error middleware

Each backend service has middleware that converts exceptions into HTTP statuses.

Common statuses:

| Status | Meaning |
|---:|---|
| `400` | Invalid or malformed request |
| `401` | Missing or invalid authentication |
| `403` | Authenticated user lacks required role |
| `404` | Resource was not found |
| `409` | Duplicate or conflicting data |
| `500` | Unexpected server failure |
| `502` | A downstream service call failed |

## 23. How to run the complete project locally

### Terminal 1: User Service

```powershell
dotnet run --project backend\services\UserService\UserService.csproj --launch-profile http
```

### Terminal 2: Identity Service

```powershell
dotnet run --project backend\services\IdentityService\IdentityService.csproj --launch-profile http
```

### Terminal 3: Property Service

```powershell
dotnet run --project backend\services\PropertyService\PropertyService.csproj --launch-profile http
```

### Terminal 4: Cart Service

```powershell
dotnet run --project backend\services\CartService\CartService.csproj --launch-profile http
```

### Terminal 5: API Gateway

```powershell
dotnet run --project backend\services\ApiGateway\ApiGateway.csproj --launch-profile http
```

### Terminal 6: Angular frontend

From the `frontend` directory:

```powershell
npm install
npm start
```

Open:

```text
http://localhost:4200
```

The backend services require the .NET 8 SDK and SQL Server LocalDB.

## 24. Complete end-to-end example: seller publishes and buyer saves

This example shows how every part connects.

```text
1. The seller registers.
   RegisterComponent
     -> Identity Service
     -> User Service profile
     -> Identity account

2. The seller logs in.
   LoginComponent
     -> Identity Service verifies password
     -> JWT is created
     -> token and profileId are saved in localStorage

3. The seller opens /seller.
   Angular role guard allows Seller role.
   Seller component loads states and existing listings.

4. The seller submits a property.
   Angular sends POST /api/properties with JWT.
   Gateway sends it to Property Service.
   Property Service reads the seller profile ID from JWT.
   Property Service resolves city/state through User Service.
   Property is saved as Pending and inactive.

5. An admin logs in.
   JWT role is Admin.

6. The admin opens /admin.
   Angular loads pending properties.
   Property Service returns Pending properties.

7. The admin verifies the property.
   PUT /api/properties/admin/{id}/verify
   Property Service sets status to Verified and active=true.

8. A buyer logs in.
   JWT role is Buyer.

9. The buyer opens /search.
   Property Service returns the newly verified property.

10. The buyer opens the details.
    Property Service returns the public property.

11. The buyer saves the property.
    POST /api/cart
    Gateway sends it to Cart Service.
    Cart Service validates the property through Property Service.
    Cart Service stores BuyerId and PropertyId.

12. The buyer opens /cart.
    Cart Service reads the buyer's saved rows.
    Cart Service retrieves current property details.
    Angular displays the saved home.
```

## 25. Quick file guide

| Area | Important file |
|---|---|
| Frontend startup | `src/main.ts` |
| Root UI layout | `src/app/app.ts` |
| Frontend routes | `src/app/app.routes.ts` |
| HTTP configuration | `src/app/app.config.ts` |
| API address | `src/app/core/api.config.ts` |
| Login/session logic | `src/app/core/services/auth.service.ts` |
| Property/cart API calls | `src/app/core/services/property.service.ts` |
| Location API calls | `src/app/core/services/location.service.ts` |
| Role guard | `src/app/core/guards/role.guard.ts` |
| Token interceptor | `src/app/core/interceptors/auth.interceptor.ts` |
| Error interceptor | `src/app/core/interceptors/error.interceptor.ts` |
| Shared TypeScript models | `src/app/core/models/ehs.models.ts` |
| Gateway routing | `backend/services/ApiGateway/src/RouteServiceResolver.cs` |
| Gateway startup | `backend/services/ApiGateway/src/Program.cs` |
| Identity endpoints | `backend/services/IdentityService/src/controllers/IdentityController.cs` |
| User/location endpoints | `backend/services/UserService/src/controllers/UsersController.cs` |
| Property endpoints | `backend/services/PropertyService/src/controllers/PropertiesController.cs` |
| Cart endpoints | `backend/services/CartService/src/controllers/CartController.cs` |

## 26. Main idea to remember

The complete request pattern is always approximately:

```text
Page
  -> Angular service
  -> HTTP interceptor
  -> API gateway
  -> correct backend controller
  -> application service
  -> repository
  -> database
  -> application service
  -> controller
  -> gateway
  -> Angular component
  -> updated page
```

The frontend decides what the person sees. The backend decides whether the requested operation is valid and permitted.
