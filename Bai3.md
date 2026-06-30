1.  Prompt đã thiết kế
    Act as a Senior System Analyst expert in E-commerce Business Logic and Software Quality Assurance.

    You are reviewing the Cart Module within the `Guai-api` codebase, specifically the `CartService.java` implementation for adding items to the cart. The current code is provided below:

    @Service
    public class CartService {
    @Autowired
    private CartRepository cartRepository;
    @Autowired
    private ProductRepository productRepository;

        public void addToCart(Long userId, Long productId, int quantity) {
            Product product = productRepository.findById(productId)
                .orElseThrow(() -> new ResourceNotFoundException("Product not found"));

            CartItem item = cartRepository.findByUserIdAndProductId(userId, productId);
            if (item != null) {
                item.setQuantity(item.getQuantity() + quantity);
            } else {
                item = new CartItem(userId, productId, quantity);
            }
            cartRepository.save(item);
        }

    }

2.  Danh sách Business Rules

    ## 1. Vulnerability Analysis

    Based on a technical review of the `CartService.java` implementation, the code exhibits severe business logic vulnerabilities that expose the application to data corruption, inventory bypass, and client-side exploits:
    - **Lack of Inventory Verification (Over-selling Vulnerability):** The method successfully retrieves the `Product` entity from the database via `productRepository.findById(productId)` but fails to perform any verification on the available stock levels (e.g., checking against a field like `product.getStockQuantity()`). A client could request a quantity of 10,000 for an item that only has 3 units in stock, and the system would persist this invalid state to the database. This shifts the failure risk down the line to the critical checkout phase.
    - **Negative/Zero Quantity Injection (State Corruption):** The primitive parameter `int quantity` accepts any integer value.
    - **Negative Inputs:** If a user passes a negative integer (e.g., `-5`) for an item already in their cart, the line `item.setQuantity(item.getQuantity() + quantity)` acts as a stealthy subtraction mechanism. If the resulting value is less than zero, it corrupts the database state with a negative quantity. If it's a new item, a `CartItem` record with a negative quantity is generated.
    - **Zero Inputs:** Passing a quantity of `0` results in redundant database write operations (`cartRepository.save(item)`) that yield zero structural value, wasting database IOPS (Input/Output Operations Per Second).

    - **Accumulated Quantity Overflow:** When an item already exists in the cart, the system increments the value blindly: `item.getQuantity() + quantity`. Without an upper boundary restriction, a malicious or automated actor could make repeating API requests to overflow database limits, break front-end rendering engines, or hoard limited promotional items.

    ***

    ## 2. Formal Business Rules Specification (SRS Format)

    To mitigate these flaws, the development team must implement a robust validation filter in the service layer before any state persistence occurs. The following formal business rules must serve as the technical implementation blueprint:

    ### BR-CART-001: Strict Positivity Constraint
    - **Description:** The input parameter representing the requested product volume must be a strictly positive value.
    - **Condition/Constraint:** `quantity` must be greater than zero ($quantity > 0$).
    - **System Exception Behavior:** If the incoming `quantity` is less than or equal to zero, the system must immediately halt processing, reject the database transaction transaction-wide, and throw an `InvalidQuantityException` (mapped to HTTP Status `400 Bad Request`) returning the structural message: _"The requested item quantity must be a positive integer greater than zero."_

    ### BR-CART-002: Real-time Stock Availability Lock
    - **Description:** The total cumulative quantity of a given product within a user's shopping cart must never exceed the active warehouse inventory level for that specific product identification key.
    - **Condition/Constraint:** $\text{Target Quantity} \le \text{Product.getAvailableStock()}$
    - _Where $\text{Target Quantity} = \text{Incoming Quantity}$ (for new items) OR $(\text{Current Cart Item Quantity} + \text{Incoming Quantity})$ (for existing items)._

    - **System Exception Behavior:** If the target calculation surpasses available inventory levels, the operation must fail immediately. The transaction must roll back, and the system must throw an `InsufficientStockException` (mapped to HTTP Status `409 Conflict`) returning the structural message: _"Cannot add item. The total quantity requested exceeds the maximum available store inventory."_

    ### BR-CART-003: Absolute Basket Item Threshold Cap
    - **Description:** To prevent distributed denial-of-service simulation on warehouse inventories and stop automated scrapers, the system must enforce a hard constraint on the total units allowed per product line item inside an individual user basket.
    - **Condition/Constraint:** $\text{Target Quantity} \le 50$
    - **System Exception Behavior:** If the mathematical addition causes the line item total to exceed 50 units, the system must intercept the save sequence, discard mutations, and throw a `CartLimitExceededException` (mapped to HTTP Status `422 Unprocessable Entity`) returning the structural message: _"Maximum cart allowance reached. Individual product lines are capped at 50 units per user profile."_

    ### BR-CART-004: Deduplication & Idempotent Mutation
    - **Description:** The shopping cart persistence layer must maintain an identical, single-row mapping structure for each distinct `(userId, productId)` composite pairing to eliminate duplicate grid items.
    - **Condition/Constraint:** Database level unique constraint or index tracking `unique(user_id, product_id)`.
    - **System Success Behavior:** If a search on `cartRepository.findByUserIdAndProductId(userId, productId)` returns a valid reference, the system must modify the active row entity fields state-side (`existingItem.quantity += incomingQuantity`) only **after** verifying complete compliance with rules **BR-CART-002** and **BR-CART-003**. If no reference is discovered, a new entity mapping block must be initialized.
