# High-Level System Design for Marcus' E-Commerce Platform

## Overview
This document outlines the architecture and key design decisions for an e-commerce platform that allows Marcus, a bicycle shop owner, to sell customizable bicycles online. The platform is built with a split frontend and backend, communicating via an API.

## Tech Stack
* **Frontend:** React (Next.js) with Tailwind CSS  
* **Backend:** Node.js with Express  
* **Database:** MYSql or PostgreSQL (relational)  
* **API:** REST or GraphQL  
* **Authentication:** JWT-based authentication for admin users (Marcus)  
* **Hosting:** Vercel for frontend, AWS/GCP for backend and database  

## Data Model
* Products (Not using Bikes since we might want to scale this on the future)  
   * id: Unique identifier  
   * name: Name of the product  
   * description: Product details  
   * base\_price: The starting price before customizations  
   * available\_tags: List of tags (ID) compatibles with this kind of product  
   * incompatible\_tags: List of tags (ID) compatible with this kind of product  
   * stock\_status: Available, out of stock  
   * tags: List of tags (ID) that this product belongs to  
   * image: link to the URL where the image is for this product  
     <!-- For a future we could add a TYPE or CATEGORY and add different products  -->
*  Parts  
   * id: Unique identifier  
   * name: Part name (e.g., "Frame" or “Frame matte finish”)  
   * stock\_status: Available, out of stock  
   * group\_id: ID of the Group Part  
   * price: Price for this particular option  
   * tags: List of tags (ID) that this part belongs to  
   * incompatibility\_list: List of tags that are incompatible with this part  
   * image: link to the URL where the image is for this part  
*  Parts Group  
    * id: Unique identifier  
    * name: Part group name (e.g., "Frame" or “Frame matte finish”)  
    * description: Description of this group  
*  Tags  
    * id: Unique identifier  
    * name: Tag name (eg., “Carbon”)  
    * order: Let’s make an order so we could have priority-parts  
    * description: Description of the tag  
      <!-- For a future we could add a TYPE or CATEGORY and add different products  -->
*  Orders & Cart (1 product with 0 or many parts per order and cart)  
    * id: Unique identifier  
    * user\_id: Reference to the user who placed the order  
    * parts: List of selected parts  
    * product: List of selected product  
    * total\_price: Final computed price  
    * created\_at: Timestamp when it was created  

## Main user actions:  
### Browsing and Customizing a Bicycle
* The customer visits the website and navigates to the product page.  
* The system loads the available bicycle models and their customization options.  
* As the user selects parts (eg: frame, wheels, rim color), the system:  
    * Calculates and updates the total price in real-time.  
    * Disables or marks unavailable parts as “out of stock.”  
### Adding a Customized Bicycle to the Cart  
* Once satisfied with the customization, the user clicks “Add to Cart.”  
* The backend:  
    * Validates the selected configuration (ensures valid part combinations with the base product or bicycle).  
    * Checks stock availability.  
    * Stores the custom selection in the cart database with a unique ID.  
* The user can view the cart with itemized pricing before proceeding to checkout.  
### Managing the Shopping Cart  
**The user can**:  
   * View all added custom bicycles with their configurations.  
   * Remove or modify selections before placing an order.  
   * Proceed to checkout when ready.  
### Placing an Order  
* The user confirms their cart and proceeds to checkout:  
    * Enters payment and shipping details .
    <!-- What about Fullfilment ? Do we assume that it’s for pick-up at the store ?   -->
    * The system verifies stock one final time.  
    * If valid, the order is stored in the Orders table, and payment is processed.  
    * The user receives an order confirmation email.  
## Product page:  
* **UI Presentation**  
    The product (detail) page should be an intuitive interface that guides the customer through the customization process  
    * Key elements
        * **Product Overview**
            * Display the selected bicycle, with the base price, description, availability(stock) and customization options (which for the sake of simplicity are just other parts)
            * Showcase the bicycle with a large high quality image and below suggest other similar bikes  
        * **Customization section**
            * As mentioned before, a customization section with each possible customization or part.  
            * Disable the parts that cannot be selected due to stock or incompatibility tags restrictions.  
        * **Pricing**
            * Update the price after each part is selected or changed  
            * Add to Cart or Buy now option  
            * Two buttons easy to see and click or press, one main one with the primary colors of the theme for the website (BUY button) and anotherone for adding to cart with the secondary color as border color  
    * Mock up for the product page
    ![](https://github.com/user-attachments/assets/a0d3fd07-fe8f-4a47-9ab7-b8501f5bd6e4)
* **Calculate availability**  
    * To do so, when loading the product detail page we would call `GET /products/{id}`
    * That call in the BE would hydrate the basic product information with the potential available and not-available parts (stock since we don’t want to show the parts that are not compatible with this bike)  
        * We would call the database to retrieve all part tags in `available_tags` for that product  
        * We are going to bring all the parts that belongs to that tag  
        * We are going to flag the ones that does not fulfills the criteria on dependencies or stock and we send that to the FE so it can be displayed in the UI  
    * After selecting parts, the UI can disabled the ones that are not compatible with the current selected by filtering by `incompatible_tags`.  
* **Dynamic Pricing calculation**  
    * The formula as explained before is: \`Total Price \= Base Price \+ Sum(Selected Options' Prices)  
    * To calculate we are going to:  
        * Start with **base price**  
        * Add selected parts or customized parts  
        * Here we would display the parts selected and the customized parts as different options, so the calculation would be just adding the parts to the base price

## Add to cart action:  
* **Frontend Actions**  
    * Both the **add to cart** or the **buy now** buttons would be disabled if the product is out of stock  
    * After the user clicks **add to cart** button, an API is called `/cart (POST)` with all the cart information (product/parts, etc) creating a new cart.  
    * Once the API is getting called, we would have the buttons on *loading* status  
    * If the call was successful, and there is no changes on the cart we could disable the button again since, there are no changes and we could show an alert/toast saying that the product(bicycle was added to the cart) and we could show a button/link to go to cart and continue with checkout.  
    * If the call returned an error we could display an error on the UI explaining what happen and how to move forward.  
* **Backend actions**  
    * We need to validate the request to ensure:  
        * Selected parts/combinations are possible for that product (bike)  
        * This could be due to combination restrictions or availability (stock, available\_parts rules)  
        * Also double check if the combination of parts is okey, by checking the incompatibility\_list for each part using the TAGS which each part belongs to.
        * Calculate the total price based on the selection and pricing rules of each part  
        * Creates a new entry for this cart on the database  
        * Returns the response to the UI (Message that the item was added to the cart succesfully, total price and the cart ID to go to the cart page)  
* **Database actions**  
    The database will update the cart table with the following data:
    ```
        {
        "id": "abc123",
        "user_id": "user789",  
        "product_id": "bike456",  
        "selected_options": [”frame_1_mate”, “rim_1”],
        "total_price": 410.00  
        }
    ```
     <!-- Where ”frame_1_mate”, “rim_1” aare IDs for the parts -->

## Administrative workflows:  
* **Managing inventory**  
    Marcus logs into the admin dashboard and:  
    * Adds, removes or modify bicycle models (base).  
    * Updates available parts and their options (e.g., adds a new rim color).  
    * Connects the parts with each base bicycle.  
    * Adds, removes or modify new parts  
        * Add restrictions to each part with other parts.  
        * Marks certain parts as “out of stock.”  
* **Setting Pricing Rules**  
    * Marcus can define pricing adjustmentsfor different part combinations:  
    * Example: “Matte finish on a full-suspension frame costs €50 instead of €30.”  
* **Handling Orders**  
    * Marcus views pending orders and confirms shipments.  
    * Updates order statuses (e.g., “Processing,” “Shipped”).  
## New product creation:  
**Create new product**  
In order to create a new product, we would need the basic information for that product
```
    name,  
    description,  
    price (base),  
    product,  
    product,  
    stock_status to indicate if the product (bicycle is available or not),  
    tags: List of tags (ID) that this product belongs to  
    image: link to the URL where the image is for this product  
```
So to create a new product, Marcus would have to go to the New product page and provide those values to the UI and click the main action button (Save).
After creating a new product the database will persist this information on the product table
## Adding a new part choice:  
* **Add a new part**  
    * In order to create a new part, for example a new rim color, the process for Marcus would be just access the UI for new part choice, complete the required information for a part which are:  
        ```
        name: Part name  
        stock_status: If it is available or out of stock  
        group: There will be a select from where Marcus can add this part to a group  
        price: Price for this particular part  
        tags: List of tags (ID) that this part belongs to  
        incompatibility_list: List of tags that are incompatible with this part  
        image: link to the URL where the image is for this part  
        ```
    * In order to introduce a new rim color for eg., Red Rim 17 inch, the completed information payload that goes into the API would look something like this:  
        * name: “Red Rim 17 inch”  
        * stock\_status: 1  
        * group: “Rims”  
        * price: 13  
        * tags: RIMS  
        * incompatibility\_list: BMX  
        * image: /path/to/image  
## Setting prices:  
* To change the Price of a part or Product, Marcus will access the Part detail page or Product detail page and update it.  
    To cover for example the **Bike Matte Finish** vs the **Frame Matte Finish** restriction, as I mentioned before we are going to treat them as different customizations or parts, so it will be a **Bike Matte Finish** with a particular price and a **Frame Matte Finish** with a different price.  
* Regarding the database changes, the only change that will be impacted will be the price for each part.  
## Tags:  
* We would need a UI to cover the creation/modification of tags.  
* For creating the TAG the only thing we need is a simple UI to add them and drag and drop them to set the order.  
* To add them the only thing we are going to require is the name and the description could be an optional field.

## API Design

Endpoints

**Product Management**
```
GET /products \- Fetch all products  
GET /products/{id} \- Fetch a single product  
POST /products \- Create a new product  
PATCH /products/{id} \- Update product details
```

**Parts & Cart**
```
GET /parts/{id} \- Fetch a single part ? 
GET /parts \- Get available parts ?  
POST /cart/ \- Create new cart  
POST /cart/{id} \- Update cart  
GET /cart/{id} \- Retrieve cart details
```
**Orders**
```
POST /orders \- Place an order  
GET /orders/{id} \- Fetch order details
```
**Admin Management**
```
POST /admin/tags \- Add/update tags 
POST /admin/products \- Add/update products 
POST /admin/parts \- Manage part inventory
```

## Future Enhancements
* Implement multi-category support for skis, surfboards, etc.  
* Add discount codes and promotions.  
* Enhance search and filtering for products.  
* Introduce AI-based recommendations.
* Create your bike feature ![](https://github.com/user-attachments/assets/679f6fef-e13f-4b24-a4c2-c12c6027e79b)
