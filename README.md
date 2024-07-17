# OLTP-Banking-Database
## Project Description
In this project an OLTP (Online Transaction Processing) database for a banking system is developed using `PostgreSQL`. The primary objective is to explore and demonstrate key database concepts, such as:
* Implementing transaction management with `rollbacks`
* Creating and utilizing `triggers` for automated actions
* Developing `stored procedures` to encapsulate business logic
* Enforcing `data integrity` with constraints
* Enhancing query performance through `indexing`
## Banking System Data Model
Bank legislation varies from country to country, and not all banks offer the same products and services. In this project, we consider a simple and generic data model for a banking system. This data model represents how data is stored, organized, and accessed in a database.

<br>
<div align = center>
<img src="bank_OLTP_ERD.png" align="center" width="1000" />
</div>


### Entities and Relationships
**Department:** This entity stores information about different departments within the organization. It has a `one-to-many` relationship with the `Employee` entity, as each department can have multiple employees.

**Employee:** This entity stores information about employees. It includes a foreign key `department_id` which references the `Department` entity, establishing a relationship that employees belong to `specific departments`.

**Customer:** This entity stores information about customers. Each customer can have multiple accounts, establishing a `one-to-many` relationship with the `Account` entity.

**Account:** This entity stores information about customer accounts. It includes a foreign key `customer_id` which references the `Customer` entity. Each account is associated with `one customer`.

**Transaction:** This entity stores information about transactions. It includes foreign keys `account_id` and `employee_id`, establishing relationships with the `Account` and `Employee` entities. Each transaction is associated with a `specific account` and processed by a `specific employee`.

The [main.py](/main.py) script sets up a database according to the specified data model. It creates two schemas: `human_resource` and `finance`, and defines tables within these schemas: `department`, `employee`, `customer`, `account`, and `transaction`, with the necessary indexing and constraints. The script also inserts sample data into these tables.

## Enhancing query performance through indexing

Indexes are used to quickly locate and access data in a database table without having to search every row of the table each time it is accessed.

`B-tree` indexing, used for `FK` columns as shown below, enhances the performance of join operations, referential integrity checks, and query execution involving foreign key relationships. This leads to more efficient and scalable database applications.

    CREATE INDEX IF NOT EXISTS idx_transaction_account_id
    ON finance.transaction USING btree (account_id);

    CREATE INDEX IF NOT EXISTS idx_employee_department_id
    ON human_resource.employee USING btree (department_id);

    CREATE INDEX IF NOT EXISTS idx_account_customer_id
    ON finance.account USING btree (customer_id);

Additionally, creating a `composite index` on `(first_name, last_name)` is beneficial for queries that involve both columns together, enhancing search performance for such queries.

    CREATE INDEX IF NOT EXISTS idx_first_name_last_name
    ON human_resource.employee USING btree (first_name, last_name);

## Enforcing data integrity with constraints

In databases, constraints are rules applied to data columns to ensure data integrity, accuracy, and reliability. They restrict the type of data that can be inserted into a table to maintain consistency and correctness.

**Primary Key Constraints** :
* Primary keys ensure that each record in a table is unique.
  * `department_pkey` on `human_resource.department(department_id)`
  * `employee_pkey` on `human_resource.employee(employee_id)`
  * `user_pkey` on `finance.customer(customer_id)`
  * `account_pkey` on `finance.account(account_id)`
  * `transaction_pkey` on `finance.transaction(transaction_id)`

**Foreign Key Constraints**: 
* **Foreign keys** ensure referential integrity by requiring that a value in one table must exist in another table.
  * `department_fkey` on `human_resource.employee(department_id`) references `human_resource.department(department_id)`
  * `user_fkey` on `finance.account(customer_id)` references `finance.customer(customer_id)`
  * `account_fkey` on `finance.transaction(account_id)` references `finance.account(account_id)`
  * `employee_fkey` on `finance.transaction(employee_id)` references `human_resource.employee(employee_id)`

**Check Constraints**: 
* **Check constraints** ensure that values in a column meet specific conditions.
  * `chk_valid_customer_type` on `finance.customer(customer_type)` ensures that the value is either retail or corporate.
  * `chk_positive_balance` on `finance.account(current_balance)` ensures that the balance is non-negative.
  * `chk_non_negative_amount` on `finance.transaction(amount)` ensures that the transaction amount is positive.
  * `chk_salary_positive` on `human_resource.employee(salary)` ensures that the salary is positive.

## Implementing transaction management with rollbacks

The finance.transfer procedure encapsulates the logic for secure and reliable fund transfers between accounts (sender_id and receiver_id). It ensures `transactional integrity` by either completing all operations successfully or rolling back completely in case of failure. The procedure employs robust exception handling to manage unexpected issues and maintains `atomicity`, treating all database changes as a single unit of work, thus adhering to `ACID` properties.

        CREATE OR REPLACE PROCEDURE finance.transfer(
        	IN sender_id integer,
        	IN receiver_id integer,
        	IN amount numeric,
            IN employee_id integer
            )
        LANGUAGE 'plpgsql'
        AS $$
                DECLARE
                    rollback_message TEXT := 'Transaction rolled back: Insufficient funds';
                    commit_message TEXT := 'Transaction committed successfully';
                BEGIN
                    IF (SELECT current_balance FROM finance.account WHERE account_id = sender_id) < amount THEN
                    RAISE EXCEPTION '%', rollback_message;
                    ELSE
                        UPDATE finance.account SET current_balance = current_balance - amount WHERE account_id = sender_id;
                        UPDATE finance.account SET current_balance = current_balance + amount WHERE account_id = receiver_id;
                        INSERT INTO finance.transaction (account_id, transaction_type, amount, employee_id) VALUES (sender_id, 'WITHDRAWAL', amount, employee_id);
                        INSERT INTO finance.transaction (account_id, transaction_type, amount, employee_id) VALUES (receiver_id, 'DEPOSIT', amount, employee_id);
                    END IF;
                EXCEPTION
                    WHEN OTHERS THEN
                    -- Rollback the transaction
                    RAISE EXCEPTION 'Error transferring funds: %', SQLERRM;
                    ROLLBACK;
                END;
        $$;
        
## Creating and utilizing triggers for automated actions

Triggers are powerful tools in databases that help automate tasks, enforce data integrity, and implement complex business rules by executing predefined actions in response to specific events.

In this project, the trigger function `employee_salary_update_func()` is designed to insert a record into the `employee_salary_update` table whenever an employee's salary is updated, serving an auditing purpose.

1. **Creating the Audit Table**

* `employee_salary_update table` captures the details of any salary changes, including the old and new salaries and the date of the chan

        CREATE TABLE IF NOT EXISTS human_resource.employee_salary_update
        (
            log_id integer NOT NULL GENERATED ALWAYS AS IDENTITY,
            employee_id integer NOT NULL,
            first_name VARCHAR(50),
            last_name VARCHAR(50),
            old_salary numeric(10,2) NOT NULL,
            incremented_salary numeric(10,2) NOT NULL,
            incremented_on date NOT NULL,
            CONSTRAINT employee_salary_update_pkey PRIMARY KEY (log_id)
        );

2. **Creating the Trigger Function**

* This function compares the old and new salary values. If they are different, it logs the change in the `employee_salary_update table`.

        CREATE OR REPLACE FUNCTION human_resource.employee_salary_update_func()
            RETURNS trigger
            LANGUAGE plpgsql
            AS $$
                BEGIN
                    IF NEW.salary <> OLD.salary THEN
                        INSERT INTO human_resource.employee_salary_update (employee_id, first_name, last_name, old_salary, incremented_salary, incremented_on)
                        VALUES (OLD.employee_id, OLD.first_name, OLD.last_name, OLD.salary, NEW.salary, now());
                    END IF;
                    RETURN NEW;
                END;
            $$;

3. **Creating the Trigger**

* This trigger ensures that the `employee_salary_update_func` function is called automatically whenever the salary column of the employee table is updated.

        CREATE TRIGGER employee_salary_update_trigger
            AFTER UPDATE OF salary ON human_resource.employee
            FOR EACH ROW
            EXECUTE FUNCTION human_resource.employee_salary_update_func();

4. **Executing an Update to Fire the Trigger**

* This update statement triggers the `employee_salary_update_func` function, which logs the salary change in the `employee_salary_update` table.

        UPDATE human_resource.employee
        SET salary = 90000
        WHERE first_name = 'John' AND last_name = 'Doe';



