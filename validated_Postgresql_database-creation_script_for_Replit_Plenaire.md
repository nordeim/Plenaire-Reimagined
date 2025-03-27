Here are the key changes incorporated:

1.  **`carts.user_id` Uniqueness Removed:** The `UNIQUE` constraint has been removed from the `user_id` column in the `carts` table definition, aligning it precisely with the schema description (`user_id INTEGER NOT NULL`).
2.  **`wishlists` Uniqueness Confirmation:** The script already had the potential `UNIQUE` constraint for `(user_id, product_id)` in `wishlists` commented out. This has been double-checked and remains commented out, matching the requirement that duplicates are allowed.
3.  **Explicit Indexing Note:** A comment has been added acknowledging that the script creates indexes implicitly via Primary Keys, Foreign Keys, and Unique Constraints as defined in the *provided* schema description, but does not add *additional* explicit performance indexes (like `CREATE INDEX ...`) as those were not specified in the initial schema details.

Here is the revised Python script:

```python
import psycopg2
import os
from dotenv import load_dotenv

# --- Configuration ---
# Load environment variables from a .env file if it exists
# Create a .env file in the same directory with:
# DB_NAME=your_database_name
# DB_USER=your_username
# DB_PASSWORD=your_password
# DB_HOST=localhost
# DB_PORT=5432
load_dotenv()

DB_CONFIG = {
    "database": os.getenv("DB_NAME", "plenaire_ecommerce"), # Default DB name
    "user": os.getenv("DB_USER", "postgres"),       # Default user
    "password": os.getenv("DB_PASSWORD", ""),       # Default empty password
    "host": os.getenv("DB_HOST", "localhost"),      # Default host
    "port": os.getenv("DB_PORT", "5432")            # Default port
}

# --- SQL Definitions ---

# Enumerated Types
SQL_CREATE_ENUMS = [
    """
    DO $$ BEGIN
        CREATE TYPE user_role AS ENUM ('user', 'admin');
    EXCEPTION
        WHEN duplicate_object THEN null;
    END $$;
    """,
    """
    DO $$ BEGIN
        CREATE TYPE order_status AS ENUM ('pending', 'processing', 'shipped', 'delivered', 'cancelled');
    EXCEPTION
        WHEN duplicate_object THEN null;
    END $$;
    """,
    """
    DO $$ BEGIN
        CREATE TYPE payment_status AS ENUM ('pending', 'processing', 'completed', 'failed', 'refunded');
    EXCEPTION
        WHEN duplicate_object THEN null;
    END $$;
    """
]

# Table Creation Statements
# Using SERIAL for auto-incrementing integer primary keys
# Using TIMESTAMP WITH TIME ZONE for timestamp columns as generally recommended in Postgres
# Using TEXT for variable length strings
SQL_CREATE_TABLES = [
    """
    CREATE TABLE IF NOT EXISTS users (
        id SERIAL PRIMARY KEY,
        email TEXT NOT NULL UNIQUE,
        password TEXT NOT NULL,
        first_name TEXT NULL,
        last_name TEXT NULL,
        phone TEXT NULL,
        role user_role NULL DEFAULT 'user',
        created_at TIMESTAMP WITH TIME ZONE NULL DEFAULT now(),
        updated_at TIMESTAMP WITH TIME ZONE NULL DEFAULT now()
    );
    """,
    """
    CREATE TABLE IF NOT EXISTS sessions (
        id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
        user_id INTEGER NOT NULL,
        expires_at TIMESTAMP WITH TIME ZONE NOT NULL,
        created_at TIMESTAMP WITH TIME ZONE NULL DEFAULT now()
    );
    """,
    """
    CREATE TABLE IF NOT EXISTS addresses (
        id SERIAL PRIMARY KEY,
        user_id INTEGER NOT NULL,
        address_line1 TEXT NOT NULL,
        address_line2 TEXT NULL,
        city TEXT NOT NULL,
        state TEXT NOT NULL,
        postal_code TEXT NOT NULL,
        country TEXT NOT NULL,
        is_default BOOLEAN NULL DEFAULT false,
        created_at TIMESTAMP WITH TIME ZONE NULL DEFAULT now(),
        updated_at TIMESTAMP WITH TIME ZONE NULL DEFAULT now()
    );
    """,
    """
    CREATE TABLE IF NOT EXISTS categories (
        id SERIAL PRIMARY KEY,
        name TEXT NOT NULL,
        description TEXT NULL,
        image_url TEXT NULL,
        created_at TIMESTAMP WITH TIME ZONE NULL DEFAULT now(),
        updated_at TIMESTAMP WITH TIME ZONE NULL DEFAULT now()
    );
    """,
    """
    CREATE TABLE IF NOT EXISTS products (
        id SERIAL PRIMARY KEY,
        name TEXT NOT NULL,
        price DOUBLE PRECISION NOT NULL,
        description TEXT NOT NULL,
        image_url TEXT NOT NULL,
        featured BOOLEAN NULL DEFAULT false,
        review_count INTEGER NULL DEFAULT 0,
        ingredients TEXT[] NULL,
        category_id INTEGER NULL,
        stock INTEGER NULL DEFAULT 100,
        sku TEXT NOT NULL UNIQUE,
        created_at TIMESTAMP WITH TIME ZONE NULL DEFAULT now(),
        updated_at TIMESTAMP WITH TIME ZONE NULL DEFAULT now()
    );
    """,
    """
    CREATE TABLE IF NOT EXISTS lifestyle_items (
        id SERIAL PRIMARY KEY,
        title TEXT NOT NULL,
        description TEXT NOT NULL,
        image_url TEXT NOT NULL,
        link TEXT NOT NULL,
        created_at TIMESTAMP WITH TIME ZONE NULL DEFAULT now(),
        updated_at TIMESTAMP WITH TIME ZONE NULL DEFAULT now()
    );
    """,
    """
    CREATE TABLE IF NOT EXISTS reviews (
        id SERIAL PRIMARY KEY,
        user_id INTEGER NOT NULL,
        product_id INTEGER NOT NULL,
        rating INTEGER NOT NULL,
        comment TEXT NULL,
        created_at TIMESTAMP WITH TIME ZONE NULL DEFAULT now(),
        updated_at TIMESTAMP WITH TIME ZONE NULL DEFAULT now()
    );
    """,
    """
    CREATE TABLE IF NOT EXISTS wishlists (
        id SERIAL PRIMARY KEY,
        user_id INTEGER NOT NULL,
        product_id INTEGER NOT NULL,
        created_at TIMESTAMP WITH TIME ZONE NULL DEFAULT now()
        -- As per schema requirements, duplicates (same user_id, product_id) are allowed.
        -- Therefore, the following constraint is intentionally NOT added:
        -- -- ALTER TABLE wishlists ADD CONSTRAINT unique_user_product_wishlist UNIQUE (user_id, product_id);
    );
    """,
    """
    CREATE TABLE IF NOT EXISTS carts (
        id SERIAL PRIMARY KEY,
        user_id INTEGER NOT NULL, -- Removed UNIQUE constraint as per feedback to match schema.
        created_at TIMESTAMP WITH TIME ZONE NULL DEFAULT now(),
        updated_at TIMESTAMP WITH TIME ZONE NULL DEFAULT now()
        -- Note: Schema shows user_id NOT NULL, but doesn't specify unique.
        -- This allows scenarios like user having multiple carts (though FK ensures user exists).
        -- If only one cart per user is logically desired, application logic must handle it,
        -- or the schema should be updated to add a UNIQUE constraint here.
    );
    """,
    """
    CREATE TABLE IF NOT EXISTS cart_items (
        id SERIAL PRIMARY KEY,
        cart_id INTEGER NOT NULL,
        product_id INTEGER NOT NULL,
        quantity INTEGER NOT NULL DEFAULT 1,
        created_at TIMESTAMP WITH TIME ZONE NULL DEFAULT now(),
        updated_at TIMESTAMP WITH TIME ZONE NULL DEFAULT now()
        -- Consider adding a UNIQUE constraint on (cart_id, product_id) if a product
        -- should only appear once per cart (quantity handles multiples).
        -- -- ALTER TABLE cart_items ADD CONSTRAINT unique_cart_product UNIQUE (cart_id, product_id);
    );
    """,
    """
    CREATE TABLE IF NOT EXISTS orders (
        id SERIAL PRIMARY KEY,
        user_id INTEGER NOT NULL,
        order_number TEXT NOT NULL UNIQUE,
        status order_status NULL DEFAULT 'pending',
        total DOUBLE PRECISION NOT NULL,
        shipping_address_id INTEGER NOT NULL,
        billing_address_id INTEGER NOT NULL,
        payment_status payment_status NULL DEFAULT 'pending',
        stripe_payment_intent_id TEXT NULL,
        notes TEXT NULL,
        created_at TIMESTAMP WITH TIME ZONE NULL DEFAULT now(),
        updated_at TIMESTAMP WITH TIME ZONE NULL DEFAULT now()
    );
    """,
    """
    CREATE TABLE IF NOT EXISTS order_items (
        id SERIAL PRIMARY KEY,
        order_id INTEGER NOT NULL,
        product_id INTEGER NOT NULL,
        quantity INTEGER NOT NULL,
        price DOUBLE PRECISION NOT NULL,
        created_at TIMESTAMP WITH TIME ZONE NULL DEFAULT now()
        -- Note: updated_at was not in the original schema description for this table
    );
    """,
    """
    CREATE TABLE IF NOT EXISTS newsletter_subscriptions (
        id SERIAL PRIMARY KEY,
        email TEXT NOT NULL UNIQUE,
        created_at TIMESTAMP WITH TIME ZONE NULL DEFAULT now()
    );
    """,
    """
    CREATE TABLE IF NOT EXISTS enquiries (
        id SERIAL PRIMARY KEY,
        name TEXT NOT NULL,
        email TEXT NOT NULL,
        phone TEXT NULL,
        subject TEXT NOT NULL,
        message TEXT NOT NULL,
        user_id INTEGER NULL, -- Optional FK
        is_resolved BOOLEAN NULL DEFAULT false,
        created_at TIMESTAMP WITH TIME ZONE NULL DEFAULT now()
        -- Note: updated_at was not in the original schema description for this table
    );
    """
]

# Foreign Key Constraints
# Added separately for clarity and to ensure tables exist first
SQL_ADD_FOREIGN_KEYS = [
    # sessions -> users
    "ALTER TABLE sessions ADD CONSTRAINT fk_sessions_user FOREIGN KEY (user_id) REFERENCES users (id) ON DELETE CASCADE;",
    # addresses -> users
    "ALTER TABLE addresses ADD CONSTRAINT fk_addresses_user FOREIGN KEY (user_id) REFERENCES users (id) ON DELETE CASCADE;",
    # products -> categories
    "ALTER TABLE products ADD CONSTRAINT fk_products_category FOREIGN KEY (category_id) REFERENCES categories (id) ON DELETE SET NULL;", # Allows category deletion without losing product
    # reviews -> users, products
    "ALTER TABLE reviews ADD CONSTRAINT fk_reviews_user FOREIGN KEY (user_id) REFERENCES users (id) ON DELETE CASCADE;",
    "ALTER TABLE reviews ADD CONSTRAINT fk_reviews_product FOREIGN KEY (product_id) REFERENCES products (id) ON DELETE CASCADE;",
    # wishlists -> users, products
    "ALTER TABLE wishlists ADD CONSTRAINT fk_wishlists_user FOREIGN KEY (user_id) REFERENCES users (id) ON DELETE CASCADE;",
    "ALTER TABLE wishlists ADD CONSTRAINT fk_wishlists_product FOREIGN KEY (product_id) REFERENCES products (id) ON DELETE CASCADE;",
    # carts -> users
    "ALTER TABLE carts ADD CONSTRAINT fk_carts_user FOREIGN KEY (user_id) REFERENCES users (id) ON DELETE CASCADE;",
    # cart_items -> carts, products
    "ALTER TABLE cart_items ADD CONSTRAINT fk_cart_items_cart FOREIGN KEY (cart_id) REFERENCES carts (id) ON DELETE CASCADE;", # If cart deleted, items are deleted
    "ALTER TABLE cart_items ADD CONSTRAINT fk_cart_items_product FOREIGN KEY (product_id) REFERENCES products (id) ON DELETE CASCADE;", # If product deleted, remove from carts
    # orders -> users, addresses (shipping), addresses (billing)
    "ALTER TABLE orders ADD CONSTRAINT fk_orders_user FOREIGN KEY (user_id) REFERENCES users (id) ON DELETE RESTRICT;", # Prevent user deletion if they have orders? Or SET NULL? RESTRICT is safer.
    "ALTER TABLE orders ADD CONSTRAINT fk_orders_shipping_address FOREIGN KEY (shipping_address_id) REFERENCES addresses (id) ON DELETE RESTRICT;", # Prevent deleting address if used in order
    "ALTER TABLE orders ADD CONSTRAINT fk_orders_billing_address FOREIGN KEY (billing_address_id) REFERENCES addresses (id) ON DELETE RESTRICT;", # Prevent deleting address if used in order
    # order_items -> orders, products
    "ALTER TABLE order_items ADD CONSTRAINT fk_order_items_order FOREIGN KEY (order_id) REFERENCES orders (id) ON DELETE CASCADE;", # If order deleted, delete its items
    "ALTER TABLE order_items ADD CONSTRAINT fk_order_items_product FOREIGN KEY (product_id) REFERENCES products (id) ON DELETE RESTRICT;", # Prevent deleting product if ordered
    # enquiries -> users
    "ALTER TABLE enquiries ADD CONSTRAINT fk_enquiries_user FOREIGN KEY (user_id) REFERENCES users (id) ON DELETE SET NULL;" # Set user_id to NULL if user is deleted
]

# Note on Indexes:
# The Primary Key, Foreign Key, and Unique constraints defined above implicitly create
# indexes in PostgreSQL for efficient lookups and enforcement.
# The original schema description did not specify any *additional* explicit indexes
# (e.g., CREATE INDEX idx_name ON table_name (column_name);) for performance tuning
# on non-key columns. If such indexes are needed based on query patterns in production,
# they should be added separately. For example:
# SQL_ADD_INDEXES = [
#     "CREATE INDEX IF NOT EXISTS idx_products_name ON products (name);",
#     "CREATE INDEX IF NOT EXISTS idx_orders_created_at ON orders (created_at DESC);"
# ]
# These would be executed after table creation.


def create_schema():
    """Connects to the database and creates the schema."""
    conn = None
    cursor = None
    try:
        print(f"Connecting to database '{DB_CONFIG['database']}' on {DB_CONFIG['host']}...")
        conn = psycopg2.connect(**DB_CONFIG)
        cursor = conn.cursor()
        print("Connection successful.")

        # Create Enumerated Types
        print("\nCreating enumerated types...")
        for sql in SQL_CREATE_ENUMS:
            print(f"Executing: {sql.strip()[:100]}...") # Print start of SQL
            cursor.execute(sql)
        conn.commit() # Commit after creating types separately might be safer
        print("Enumerated types created (or already exist).")

        # Create Tables
        print("\nCreating tables...")
        for sql in SQL_CREATE_TABLES:
            print(f"Executing: CREATE TABLE IF NOT EXISTS {sql.split('(')[0].split()[-1]}...")
            cursor.execute(sql)
        # No commit here yet, do it after constraints for atomicity of table+constraints

        print("Tables created (or already exist).")

        # Add Foreign Key Constraints
        print("\nAdding foreign key constraints...")
        for sql in SQL_ADD_FOREIGN_KEYS:
            constraint_name = sql.split("ADD CONSTRAINT ")[1].split(" ")[0]
            try:
                print(f"Adding constraint: {constraint_name}...")
                cursor.execute(sql)
                print(f"  -> Added {constraint_name}.")
            except psycopg2.Error as e:
                # Error code '42P07' is duplicate_object (constraint already exists)
                # Error code '42710' is duplicate_object (index related to constraint already exists)
                if e.pgcode in ('42P07', '42710'):
                    print(f"  -> Constraint or related index {constraint_name} already exists, skipping.")
                    conn.rollback() # Rollback the individual failed ALTER TABLE
                    conn.autocommit = True # Temporarily enable autocommit to continue loop after rollback
                    cursor = conn.cursor() # Need a new cursor after rollback in some modes
                    conn.autocommit = False
                else:
                    print(f"  -> Error adding constraint {constraint_name}: {e}")
                    raise # Re-raise other errors

        print("Foreign key constraints added (or already exist).")

        # Add Explicit Unique Constraints (Not needed as handled inline)
        # If SQL_ADD_UNIQUE_CONSTRAINTS were used, add loop here similarly to FKs.

        # Add Explicit Indexes (If defined in SQL_ADD_INDEXES)
        # Example:
        # print("\nAdding explicit performance indexes...")
        # for sql in SQL_ADD_INDEXES: # Assuming SQL_ADD_INDEXES list exists
        #     index_name = sql.split("CREATE INDEX ")[1].split(" ON ")[0] # Basic name parsing
        #     try:
        #         print(f"Adding index: {index_name}...")
        #         cursor.execute(sql)
        #         print(f"  -> Added {index_name}.")
        #     except psycopg2.Error as e:
        #          if e.pgcode == '42P07': # Index already exists
        #              print(f"  -> Index {index_name} already exists, skipping.")
        #              conn.rollback()
        #              conn.autocommit = True
        #              cursor = conn.cursor()
        #              conn.autocommit = False
        #          else:
        #              print(f"  -> Error adding index {index_name}: {e}")
        #              raise # Re-raise other errors
        # print("Explicit indexes added (or already exist).")


        # Commit changes
        conn.commit()
        print("\nSchema creation process completed successfully. Changes committed.")

    except psycopg2.Error as e:
        print(f"\nDatabase error: {e}")
        if conn:
            conn.rollback() # Rollback changes on error
        print("Schema creation failed. Changes rolled back.")
    except Exception as e:
        print(f"\nAn unexpected error occurred: {e}")
        if conn:
            conn.rollback()
        print("Schema creation failed due to unexpected error. Changes rolled back.")

    finally:
        if cursor:
            cursor.close()
            print("Cursor closed.")
        if conn:
            conn.close()
            print("Database connection closed.")

if __name__ == "__main__":
    # Optional: Add check for required environment variables
    if not DB_CONFIG["password"] and not os.getenv("PGPASSWORD"): # Check env var PGPASSWORD too
       print("Warning: DB_PASSWORD environment variable not set (or PGPASSWORD). Connection might fail if password is required.")
       # Consider prompting or exiting if password is required and not found

    create_schema()

```

---

## Line-by-Line Validation Against Schema Description

Here's a comparison between the provided schema description and the generated Python script's SQL commands:

**1. Enumerations:**

*   **Schema:** `user_role` (user, admin), `order_status` (pending, processing, shipped, delivered, cancelled), `payment_status` (pending, processing, completed, failed, refunded).
*   **Script (`SQL_CREATE_ENUMS`):** Correctly creates these three ENUM types with the specified values using `CREATE TYPE ... AS ENUM (...)`. Includes `DO $$ ... EXCEPTION ...` block to prevent errors if types already exist.
*   **Result:** Matches.

**2. Tables:**

*   **`users` Table:**
    *   **Schema:** `id`(int, PK, auto-inc), `email`(text, unique), `password`(text), `first_name`(text, null), `last_name`(text, null), `phone`(text, null), `role`(user_role, default 'user'), `created_at`(ts, default now), `updated_at`(ts, default now).
    *   **Script:** `id SERIAL PRIMARY KEY`, `email TEXT NOT NULL UNIQUE`, `password TEXT NOT NULL`, `first_name TEXT NULL`, `last_name TEXT NULL`, `phone TEXT NULL`, `role user_role NULL DEFAULT 'user'`, `created_at TIMESTAMP WITH TIME ZONE NULL DEFAULT now()`, `updated_at TIMESTAMP WITH TIME ZONE NULL DEFAULT now()`. (`SERIAL` handles PK/auto-inc, `NOT NULL` added where Nullable=NO, `UNIQUE` on email).
    *   **Result:** Matches. (Note: Script uses `TIMESTAMP WITH TIME ZONE` which is generally preferred over `TIMESTAMP`.)

*   **`sessions` Table:**
    *   **Schema:** `id`(uuid, PK, default gen_random_uuid), `user_id`(int, FK), `expires_at`(ts), `created_at`(ts, default now).
    *   **Script:** `id UUID PRIMARY KEY DEFAULT gen_random_uuid()`, `user_id INTEGER NOT NULL`, `expires_at TIMESTAMP WITH TIME ZONE NOT NULL`, `created_at TIMESTAMP WITH TIME ZONE NULL DEFAULT now()`.
    *   **Result:** Matches.

*   **`addresses` Table:**
    *   **Schema:** `id`(int, PK, auto-inc), `user_id`(int, FK), `address_line1`(text), `address_line2`(text, null), `city`(text), `state`(text), `postal_code`(text), `country`(text), `is_default`(bool, default false), `created_at`(ts, default now), `updated_at`(ts, default now).
    *   **Script:** `id SERIAL PRIMARY KEY`, `user_id INTEGER NOT NULL`, `address_line1 TEXT NOT NULL`, `address_line2 TEXT NULL`, `city TEXT NOT NULL`, `state TEXT NOT NULL`, `postal_code TEXT NOT NULL`, `country TEXT NOT NULL`, `is_default BOOLEAN NULL DEFAULT false`, `created_at TIMESTAMP WITH TIME ZONE NULL DEFAULT now()`, `updated_at TIMESTAMP WITH TIME ZONE NULL DEFAULT now()`.
    *   **Result:** Matches.

*   **`categories` Table:**
    *   **Schema:** `id`(int, PK, auto-inc), `name`(text), `description`(text, null), `image_url`(text, null), `created_at`(ts, default now), `updated_at`(ts, default now).
    *   **Script:** `id SERIAL PRIMARY KEY`, `name TEXT NOT NULL`, `description TEXT NULL`, `image_url TEXT NULL`, `created_at TIMESTAMP WITH TIME ZONE NULL DEFAULT now()`, `updated_at TIMESTAMP WITH TIME ZONE NULL DEFAULT now()`.
    *   **Result:** Matches.

*   **`products` Table:**
    *   **Schema:** `id`(int, PK, auto-inc), `name`(text), `price`(double precision), `description`(text), `image_url`(text), `featured`(bool, default false), `review_count`(int, default 0), `ingredients`(text[], null), `category_id`(int, FK, null), `stock`(int, default 100), `sku`(text, unique), `created_at`(ts, default now), `updated_at`(ts, default now).
    *   **Script:** `id SERIAL PRIMARY KEY`, `name TEXT NOT NULL`, `price DOUBLE PRECISION NOT NULL`, `description TEXT NOT NULL`, `image_url TEXT NOT NULL`, `featured BOOLEAN NULL DEFAULT false`, `review_count INTEGER NULL DEFAULT 0`, `ingredients TEXT[] NULL`, `category_id INTEGER NULL`, `stock INTEGER NULL DEFAULT 100`, `sku TEXT NOT NULL UNIQUE`, `created_at TIMESTAMP WITH TIME ZONE NULL DEFAULT now()`, `updated_at TIMESTAMP WITH TIME ZONE NULL DEFAULT now()`.
    *   **Result:** Matches.

*   **`lifestyle_items` Table:**
    *   **Schema:** `id`(int, PK, auto-inc), `title`(text), `description`(text), `image_url`(text), `link`(text), `created_at`(ts, default now), `updated_at`(ts, default now).
    *   **Script:** `id SERIAL PRIMARY KEY`, `title TEXT NOT NULL`, `description TEXT NOT NULL`, `image_url TEXT NOT NULL`, `link TEXT NOT NULL`, `created_at TIMESTAMP WITH TIME ZONE NULL DEFAULT now()`, `updated_at TIMESTAMP WITH TIME ZONE NULL DEFAULT now()`.
    *   **Result:** Matches.

*   **`reviews` Table:**
    *   **Schema:** `id`(int, PK, auto-inc), `user_id`(int, FK), `product_id`(int, FK), `rating`(int), `comment`(text, null), `created_at`(ts, default now), `updated_at`(ts, default now).
    *   **Script:** `id SERIAL PRIMARY KEY`, `user_id INTEGER NOT NULL`, `product_id INTEGER NOT NULL`, `rating INTEGER NOT NULL`, `comment TEXT NULL`, `created_at TIMESTAMP WITH TIME ZONE NULL DEFAULT now()`, `updated_at TIMESTAMP WITH TIME ZONE NULL DEFAULT now()`.
    *   **Result:** Matches.

*   **`wishlists` Table:**
    *   **Schema:** `id`(int, PK, auto-inc), `user_id`(int, FK), `product_id`(int, FK), `created_at`(ts, default now). (No unique constraint on user/product specified).
    *   **Script:** `id SERIAL PRIMARY KEY`, `user_id INTEGER NOT NULL`, `product_id INTEGER NOT NULL`, `created_at TIMESTAMP WITH TIME ZONE NULL DEFAULT now()`. The script explicitly notes that a `UNIQUE(user_id, product_id)` constraint is *not* added, matching the schema description and feedback.
    *   **Result:** Matches (including the lack of uniqueness).

*   **`carts` Table:**
    *   **Schema:** `id`(int, PK, auto-inc), `user_id`(int, FK), `created_at`(ts, default now), `updated_at`(ts, default now). (No unique constraint on user_id specified).
    *   **Script:** `id SERIAL PRIMARY KEY`, `user_id INTEGER NOT NULL`, `created_at TIMESTAMP WITH TIME ZONE NULL DEFAULT now()`, `updated_at TIMESTAMP WITH TIME ZONE NULL DEFAULT now()`. The `UNIQUE` keyword previously on `user_id` has been removed as requested.
    *   **Result:** Matches (specifically corrected `user_id` as per feedback).

*   **`cart_items` Table:**
    *   **Schema:** `id`(int, PK, auto-inc), `cart_id`(int, FK), `product_id`(int, FK), `quantity`(int, default 1), `created_at`(ts, default now), `updated_at`(ts, default now).
    *   **Script:** `id SERIAL PRIMARY KEY`, `cart_id INTEGER NOT NULL`, `product_id INTEGER NOT NULL`, `quantity INTEGER NOT NULL DEFAULT 1`, `created_at TIMESTAMP WITH TIME ZONE NULL DEFAULT now()`, `updated_at TIMESTAMP WITH TIME ZONE NULL DEFAULT now()`.
    *   **Result:** Matches.

*   **`orders` Table:**
    *   **Schema:** `id`(int, PK, auto-inc), `user_id`(int, FK), `order_number`(text, unique), `status`(order_status, default 'pending'), `total`(double precision), `shipping_address_id`(int, FK), `billing_address_id`(int, FK), `payment_status`(payment_status, default 'pending'), `stripe_payment_intent_id`(text, null), `notes`(text, null), `created_at`(ts, default now), `updated_at`(ts, default now).
    *   **Script:** `id SERIAL PRIMARY KEY`, `user_id INTEGER NOT NULL`, `order_number TEXT NOT NULL UNIQUE`, `status order_status NULL DEFAULT 'pending'`, `total DOUBLE PRECISION NOT NULL`, `shipping_address_id INTEGER NOT NULL`, `billing_address_id INTEGER NOT NULL`, `payment_status payment_status NULL DEFAULT 'pending'`, `stripe_payment_intent_id TEXT NULL`, `notes TEXT NULL`, `created_at TIMESTAMP WITH TIME ZONE NULL DEFAULT now()`, `updated_at TIMESTAMP WITH TIME ZONE NULL DEFAULT now()`.
    *   **Result:** Matches.

*   **`order_items` Table:**
    *   **Schema:** `id`(int, PK, auto-inc), `order_id`(int, FK), `product_id`(int, FK), `quantity`(int), `price`(double precision), `created_at`(ts, default now). (No `updated_at`).
    *   **Script:** `id SERIAL PRIMARY KEY`, `order_id INTEGER NOT NULL`, `product_id INTEGER NOT NULL`, `quantity INTEGER NOT NULL`, `price DOUBLE PRECISION NOT NULL`, `created_at TIMESTAMP WITH TIME ZONE NULL DEFAULT now()`.
    *   **Result:** Matches.

*   **`newsletter_subscriptions` Table:**
    *   **Schema:** `id`(int, PK, auto-inc), `email`(text, unique), `created_at`(ts, default now).
    *   **Script:** `id SERIAL PRIMARY KEY`, `email TEXT NOT NULL UNIQUE`, `created_at TIMESTAMP WITH TIME ZONE NULL DEFAULT now()`.
    *   **Result:** Matches.

*   **`enquiries` Table:**
    *   **Schema:** `id`(int, PK, auto-inc), `name`(text), `email`(text), `phone`(text, null), `subject`(text), `message`(text), `user_id`(int, FK, null), `is_resolved`(bool, default false), `created_at`(ts, default now). (No `updated_at`).
    *   **Script:** `id SERIAL PRIMARY KEY`, `name TEXT NOT NULL`, `email TEXT NOT NULL`, `phone TEXT NULL`, `subject TEXT NOT NULL`, `message TEXT NOT NULL`, `user_id INTEGER NULL`, `is_resolved BOOLEAN NULL DEFAULT false`, `created_at TIMESTAMP WITH TIME ZONE NULL DEFAULT now()`.
    *   **Result:** Matches.

**3. Key Relationships (Foreign Keys):**

*   **Schema:** Lists 16 specific FK relationships (e.g., `addresses.user_id` -> `users.id`, `cart_items.cart_id` -> `carts.id`, etc.).
*   **Script (`SQL_ADD_FOREIGN_KEYS`):** Contains 16 `ALTER TABLE ... ADD CONSTRAINT ... FOREIGN KEY` statements precisely matching the source table/column and target table/column specified in the schema. Appropriate `ON DELETE` actions (`CASCADE`, `SET NULL`, `RESTRICT`) are applied. Error handling for existing constraints is included.
*   **Result:** Matches.

**4. Unique Constraints:**

*   **Schema:** Lists 4 explicit unique constraints: `newsletter_subscriptions.email`, `orders.order_number`, `products.sku`, `users.email`.
*   **Script:** These 4 constraints are correctly implemented using the `UNIQUE` keyword directly within the respective `CREATE TABLE` statements. The separate `SQL_ADD_UNIQUE_CONSTRAINTS` list is effectively empty/not used, reflecting this inline definition.
*   **Result:** Matches.

**5. Implementation Notes:**

*   **Schema:** Notes on Auth flow, Stripe, Data Integrity (FKs), Order Processing (ENUMs).
*   **Script:** Uses appropriate data types (UUID, TEXT, ENUMs), includes `stripe_payment_intent_id`, defines all specified Foreign Keys, and uses ENUMs for statuses, consistent with the notes.
*   **Result:** Matches.

**6. Missing Indexes (Feedback Point):**

*   **Schema:** The *provided schema description* did not list any `CREATE INDEX` statements beyond those implied by PK, FK, and Unique constraints.
*   **Script:** The script correctly omits `CREATE INDEX` statements, matching the provided schema. A comment is added acknowledging this and suggesting how they *could* be added if the schema definition were expanded.
*   **Result:** Matches the *provided* schema; acknowledges the feedback point about potential production indexes not detailed in the input schema.

**Conclusion:** The revised Python script accurately reflects the database schema definition provided in the initial prompt, incorporating the specific corrections requested regarding `carts.user_id` and confirming the intended behavior for `wishlists`. It correctly defines all ENUMs, tables, columns, data types, nullability, defaults, primary keys, foreign keys, and unique constraints as specified.
