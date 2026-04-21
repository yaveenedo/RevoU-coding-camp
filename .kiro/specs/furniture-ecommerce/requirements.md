# Requirements Document

## Introduction

A furniture e-commerce web application built with Django as the Python web framework, Tailwind CSS for styling, Django templates for server-side rendering, and JavaScript for dynamic interactions. The app allows any visitor to browse furniture products, but restricts purchasing actions to authenticated users. Authentication is handled exclusively via Google OAuth. Logged-in users can add items to a shopping cart, complete purchases through the Midtrans payment gateway, and leave reviews on products they have purchased.

## Glossary

- **System**: The furniture e-commerce web application as a whole.
- **Django**: The Python web framework used to implement the backend, URL routing, ORM, and server-side template rendering.
- **Django_Template_Engine**: Django's built-in templating system used to render HTML pages server-side.
- **Guest**: An unauthenticated user visiting the website.
- **User**: An authenticated user who has logged in via Google OAuth.
- **Product**: A furniture item listed for sale on the platform.
- **Cart**: A temporary collection of Products a User intends to purchase.
- **Order**: A confirmed purchase transaction associated with a User.
- **Review**: A text comment submitted by a User for a Product they have purchased.
- **Google_OAuth**: The Google OAuth 2.0 authentication service used for registration and login.
- **Midtrans**: The third-party payment gateway used to process payments.
- **Auth_Service**: The component responsible for managing authentication and session state.
- **Product_Catalog**: The component responsible for displaying and managing Product listings.
- **Cart_Service**: The component responsible for managing Cart state and Cart items.
- **Order_Service**: The component responsible for creating and tracking Orders.
- **Review_Service**: The component responsible for managing Reviews.
- **Payment_Service**: The component responsible for integrating with Midtrans.

---

## Requirements

### Requirement 1: Guest Product Browsing

**User Story:** As a Guest, I want to browse furniture products without logging in, so that I can explore the catalog before deciding to register.

#### Acceptance Criteria

1. THE Product_Catalog SHALL display all available Products to any visitor regardless of authentication state.
2. WHEN a Guest navigates to the homepage, THE Product_Catalog SHALL render the full list of Products including name, image, description, and price.
3. WHEN a Guest applies a filter or search query, THE Product_Catalog SHALL return only Products matching the criteria.

---

### Requirement 2: Google OAuth Registration and Login

**User Story:** As a Guest, I want to register and log in using my Google account, so that I can access purchasing features without managing a separate password.

#### Acceptance Criteria

1. THE Auth_Service SHALL support registration and login exclusively through Google OAuth 2.0.
2. WHEN a Guest initiates login, THE Auth_Service SHALL redirect the Guest to the Google OAuth 2.0 authorization endpoint.
3. WHEN Google OAuth returns a successful authorization, THE Auth_Service SHALL create or retrieve the User account associated with the Google identity and establish an authenticated session.
4. IF Google OAuth returns an error or the Guest denies authorization, THEN THE Auth_Service SHALL redirect the Guest to the login page and display a descriptive error message.
5. WHEN a User logs out, THE Auth_Service SHALL invalidate the current session and redirect the User to the homepage.

---

### Requirement 3: Shop Button Access Control

**User Story:** As a Guest, I want to be redirected to the login page when I try to add a product to my cart, so that I understand I need an account to shop.

#### Acceptance Criteria

1. WHILE a visitor is not authenticated, THE System SHALL display the Shop (add-to-cart) button in a visible but restricted state on each Product.
2. WHEN a Guest presses the Shop button on any Product, THE System SHALL redirect the Guest to the login/register page.
3. WHILE a User is authenticated, THE System SHALL display the Shop button in a fully interactive state on each Product.
4. WHEN an authenticated User presses the Shop button on a Product, THE Cart_Service SHALL add that Product to the User's Cart.

---

### Requirement 4: Shopping Cart Management

**User Story:** As a User, I want to manage items in my shopping cart, so that I can review and adjust my selections before purchasing.

#### Acceptance Criteria

1. THE Cart_Service SHALL maintain a persistent Cart for each authenticated User across sessions.
2. WHEN a User adds a Product to the Cart, THE Cart_Service SHALL increment the quantity of that Product in the Cart by one if it already exists, or add it as a new Cart item with quantity one.
3. WHEN a User updates the quantity of a Cart item to zero, THE Cart_Service SHALL remove that item from the Cart.
4. WHEN a User removes a Cart item, THE Cart_Service SHALL delete that item from the Cart.
5. THE Cart_Service SHALL display the current Cart total price, calculated as the sum of each Cart item's unit price multiplied by its quantity.
6. IF the Cart is empty, THEN THE Cart_Service SHALL display an empty cart message and a prompt to continue browsing.

---

### Requirement 5: Midtrans Payment Integration

**User Story:** As a User, I want to pay for my cart items using a secure payment gateway, so that I can complete my purchase safely.

#### Acceptance Criteria

1. WHEN a User initiates checkout, THE Payment_Service SHALL create a Midtrans transaction token using the Cart total and User details.
2. WHEN the Midtrans transaction token is created, THE System SHALL present the Midtrans payment interface to the User.
3. WHEN Midtrans confirms a successful payment via its callback, THE Order_Service SHALL create an Order record associated with the User and mark it as paid.
4. IF Midtrans returns a failed or cancelled payment status, THEN THE Order_Service SHALL retain the Cart contents and display a payment failure message to the User.
5. WHEN an Order is created, THE Cart_Service SHALL clear all items from the User's Cart.

---

### Requirement 6: Post-Purchase Product Reviews

**User Story:** As a User, I want to write a comment on a product I have purchased, so that I can share my experience with other shoppers.

#### Acceptance Criteria

1. WHEN a User views a Product detail page, THE Review_Service SHALL display all approved Reviews for that Product including the reviewer's name and review text.
2. WHEN a User who has a completed Order containing a Product attempts to submit a Review for that Product, THE Review_Service SHALL save the Review and associate it with the User and the Product.
3. IF a User who has no completed Order containing a Product attempts to submit a Review for that Product, THEN THE Review_Service SHALL reject the submission and display a message indicating the User must purchase the Product before reviewing.
4. IF a User attempts to submit a Review with empty text, THEN THE Review_Service SHALL reject the submission and display a validation error message.
5. THE Review_Service SHALL allow each User to submit at most one Review per Product per completed Order.

---

### Requirement 7: Tech Stack Constraints

**User Story:** As a developer, I want the application built on a defined tech stack, so that the codebase is consistent and maintainable.

#### Acceptance Criteria

1. THE System SHALL implement the backend using Django as the Python web framework, including Django's ORM for database access and URL routing.
2. THE System SHALL use the Django_Template_Engine to render all HTML pages server-side.
3. THE System SHALL apply Tailwind CSS for all frontend styling.
4. THE System SHALL use JavaScript for all client-side dynamic interactions, including cart updates, payment UI rendering, and form validation feedback.
