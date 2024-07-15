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
