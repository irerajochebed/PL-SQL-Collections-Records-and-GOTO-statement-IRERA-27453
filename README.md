# PL-SQL-Collections-Records-and-GOTO-statement-IRERA-27453

# Employee_Performance_Management_System

# 1. Problem Definition
Objective:

Create a comprehensive employee performance management system that demonstrates advanced PL/SQL concepts including:

    Collections (Associative Arrays, Nested Tables, VARRAYs)

    Records (Table-based, User-defined, Cursor-based)

    GOTO Statements for controlled flow management

Business Scenario:

A company needs to automate their quarterly performance review process. The system should:

    Process multiple employees using collections

    Store complex employee data using records

    Calculate performance scores and bonuses

    Handle exceptional cases using GOTO statements

    Generate comprehensive performance reports

# 2. Database Setup
sql

-- Employee table structure
CREATE TABLE employees (
    emp_id NUMBER PRIMARY KEY,
    emp_name VARCHAR2(100),
    department VARCHAR2(50),
    salary NUMBER,
    hire_date DATE
);

-- Performance reviews table
CREATE TABLE performance_reviews (
    review_id NUMBER PRIMARY KEY,
    emp_id NUMBER,
    review_date DATE,
    technical_skill NUMBER,
    communication NUMBER,
    productivity NUMBER,
    overall_rating NUMBER,
    FOREIGN KEY (emp_id) REFERENCES employees(emp_id)
);

-- Sample data
INSERT INTO employees VALUES (1, 'Alice Johnson', 'IT', 75000, DATE '2020-01-15');
INSERT INTO employees VALUES (2, 'Bob Smith', 'HR', 65000, DATE '2019-03-20');
INSERT INTO employees VALUES (3, 'Charlie Brown', 'IT', 80000, DATE '2018-06-10');
INSERT INTO employees VALUES (4, 'Diana Prince', 'FINANCE', 90000, DATE '2021-02-28');
INSERT INTO employees VALUES (5, 'Edward Wilson', 'MARKETING', 55000, DATE '2022-11-05');

INSERT INTO performance_reviews VALUES (1, 1, DATE '2024-01-15', 8, 7, 9, 8);
INSERT INTO performance_reviews VALUES (2, 2, DATE '2024-01-16', 6, 8, 7, 7);
INSERT INTO performance_reviews VALUES (3, 3, DATE '2024-01-17', 9, 6, 8, 7.5);
INSERT INTO performance_reviews VALUES (4, 4, DATE '2024-01-18', 7, 9, 8, 8);
-- Employee 5 has no performance review (for GOTO demonstration)

COMMIT;

# 3. Complete PL/SQL Solution
sql

SET SERVEROUTPUT ON;

DECLARE
    -- ==========================================
    -- COLLECTION DECLARATIONS
    -- ==========================================
    
    -- 1. VARRAY for department bonus rates (fixed size)
    TYPE dept_bonus_array IS VARRAY(10) OF NUMBER;
    v_bonus_rates dept_bonus_array := dept_bonus_array(0.10, 0.08, 0.12, 0.09, 0.07);
    
    -- 2. Associative Array for department mapping
    TYPE dept_map IS TABLE OF VARCHAR2(50) INDEX BY VARCHAR2(10);
    v_department_codes dept_map;
    
    -- 3. Nested Table for employee records
    TYPE emp_rec_type IS RECORD (
        emp_id employees.emp_id%TYPE,
        emp_name employees.emp_name%TYPE,
        department employees.department%TYPE,
        salary employees.salary%TYPE,
        performance_rating performance_reviews.overall_rating%TYPE,
        calculated_bonus NUMBER
    );
    
    TYPE emp_table_type IS TABLE OF emp_rec_type;
    v_employees emp_table_type;
    
    -- ==========================================
    -- VARIABLE DECLARATIONS
    -- ==========================================
    v_total_bonus_payout NUMBER := 0;
    v_high_performers_count NUMBER := 0;
    v_invalid_data_found BOOLEAN := FALSE;

BEGIN
    DBMS_OUTPUT.PUT_LINE('*** EMPLOYEE PERFORMANCE MANAGEMENT SYSTEM ***');
    DBMS_OUTPUT.PUT_LINE('=============================================');
    DBMS_OUTPUT.PUT_LINE('Starting processing at: ' || TO_CHAR(SYSDATE, 'YYYY-MM-DD HH24:MI:SS'));
    DBMS_OUTPUT.PUT_LINE('');
    
    -- ==========================================
    -- INITIALIZE DEPARTMENT MAPPING (Associative Array)
    -- ==========================================
    v_department_codes('IT') := 'Information Technology';
    v_department_codes('HR') := 'Human Resources';
    v_department_codes('FINANCE') := 'Finance Department';
    v_department_codes('MARKETING') := 'Marketing Team';
    v_department_codes('SALES') := 'Sales Department';
    
    -- ==========================================
    -- BULK COLLECT EMPLOYEE DATA (Collections + Records)
    -- ==========================================
    SELECT 
        e.emp_id,
        e.emp_name,
        e.department,
        e.salary,
        NVL(p.overall_rating, 0),
        0  -- Placeholder for calculated_bonus
    BULK COLLECT INTO v_employees
    FROM employees e
    LEFT JOIN performance_reviews p ON e.emp_id = p.emp_id
    ORDER BY e.emp_id;
    
    -- ==========================================
    -- PROCESS EACH EMPLOYEE RECORD
    -- ==========================================
    FOR i IN 1..v_employees.COUNT LOOP
        DBMS_OUTPUT.PUT_LINE('--- Processing Employee: ' || v_employees(i).emp_name || ' (ID: ' || v_employees(i).emp_id || ') ---');
        
        -- ==========================================
        -- GOTO DEMONSTRATION: Handle missing performance data
        -- ==========================================
        IF v_employees(i).performance_rating = 0 THEN
            DBMS_OUTPUT.PUT_LINE('    WARNING: No performance review found for employee ' || v_employees(i).emp_name);
            DBMS_OUTPUT.PUT_LINE('   ↳ Skipping bonus calculation and moving to next employee...');
            v_invalid_data_found := TRUE;
            GOTO skip_bonus_calculation;
        END IF;
        
        -- ==========================================
        -- BONUS CALCULATION LOGIC
        -- ==========================================
        DECLARE
            v_bonus_rate NUMBER;
            v_department_index NUMBER;
        BEGIN
            -- Determine department-specific bonus rate using CASE
            CASE v_employees(i).department
                WHEN 'IT' THEN v_bonus_rate := 0.10;
                WHEN 'HR' THEN v_bonus_rate := 0.08;
                WHEN 'FINANCE' THEN v_bonus_rate := 0.12;
                WHEN 'MARKETING' THEN v_bonus_rate := 0.09;
                WHEN 'SALES' THEN v_bonus_rate := 0.15;
                ELSE 
                    DBMS_OUTPUT.PUT_LINE('   ⚠️  Unknown department: ' || v_employees(i).department);
                    v_bonus_rate := 0.05; -- Default rate
            END CASE;
            
            -- Calculate bonus based on performance rating
            v_employees(i).calculated_bonus := v_employees(i).salary * 
                                             v_bonus_rate * 
                                             (v_employees(i).performance_rating / 10);
            
            -- Add to total payout
            v_total_bonus_payout := v_total_bonus_payout + v_employees(i).calculated_bonus;
            
            -- Count high performers (rating >= 8)
            IF v_employees(i).performance_rating >= 8 THEN
                v_high_performers_count := v_high_performers_count + 1;
            END IF;
            
            -- Display calculation details
            DBMS_OUTPUT.PUT_LINE('   Department: ' || v_department_codes(v_employees(i).department));
            DBMS_OUTPUT.PUT_LINE('   Performance Rating: ' || v_employees(i).performance_rating);
            DBMS_OUTPUT.PUT_LINE('   Bonus Rate: ' || (v_bonus_rate * 100) || '%');
            DBMS_OUTPUT.PUT_LINE('   Calculated Bonus: $' || ROUND(v_employees(i).calculated_bonus, 2));
            
        END;
        
        -- ==========================================
        -- GOTO LABEL: Skip calculation for invalid data
        -- ==========================================
        <<skip_bonus_calculation>>
        NULL; -- Required NULL statement after label
        
        DBMS_OUTPUT.PUT_LINE(''); -- Empty line for readability
        
    END LOOP;
    
    -- ==========================================
    -- FINAL SUMMARY REPORT
    -- ==========================================
    DBMS_OUTPUT.PUT_LINE('=============================================');
    DBMS_OUTPUT.PUT_LINE('*** PERFORMANCE REVIEW SUMMARY ***');
    DBMS_OUTPUT.PUT_LINE('=============================================');
    DBMS_OUTPUT.PUT_LINE('Total Employees Processed: ' || v_employees.COUNT);
    DBMS_OUTPUT.PUT_LINE('High Performers (Rating ≥ 8): ' || v_high_performers_count);
    DBMS_OUTPUT.PUT_LINE('Total Bonus Payout: $' || ROUND(v_total_bonus_payout, 2));
    DBMS_OUTPUT.PUT_LINE('Average Bonus per Employee: $' || 
        ROUND(v_total_bonus_payout / NULLIF(v_employees.COUNT, 0), 2));
    
    IF v_invalid_data_found THEN
        DBMS_OUTPUT.PUT_LINE('⚠️  Note: Some employees were skipped due to missing performance data.');
    END IF;
    
    DBMS_OUTPUT.PUT_LINE('');
    DBMS_OUTPUT.PUT_LINE('Processing completed at: ' || TO_CHAR(SYSDATE, 'YYYY-MM-DD HH24:MI:SS'));
    DBMS_OUTPUT.PUT_LINE('*** SYSTEM PROCESSING COMPLETE ***');

EXCEPTION
    WHEN OTHERS THEN
        DBMS_OUTPUT.PUT_LINE('*** CRITICAL ERROR ***');
        DBMS_OUTPUT.PUT_LINE('Error Code: ' || SQLCODE);
        DBMS_OUTPUT.PUT_LINE('Error Message: ' || SQLERRM);
        DBMS_OUTPUT.PUT_LINE('*** Please contact system administrator ***');
END;
/

# 4. Expected Output
text

*** EMPLOYEE PERFORMANCE MANAGEMENT SYSTEM ***
=============================================
Starting processing at: 2025-11-25 19:30:25

--- Processing Employee: Alice Johnson (ID: 1) ---
   Department: Information Technology
   Performance Rating: 8
   Bonus Rate: 10%
   Calculated Bonus: $6000

--- Processing Employee: Bob Smith (ID: 2) ---
   Department: Human Resources
   Performance Rating: 7
   Bonus Rate: 8%
   Calculated Bonus: $3640

--- Processing Employee: Charlie Brown (ID: 3) ---
   Department: Information Technology
   Performance Rating: 7.5
   Bonus Rate: 10%
   Calculated Bonus: $6000

--- Processing Employee: Diana Prince (ID: 4) ---
   Department: Finance Department
   Performance Rating: 8
   Bonus Rate: 12%
   Calculated Bonus: $8640

--- Processing Employee: Edward Wilson (ID: 5) ---
   WARNING: No performance review found for employee Edward Wilson
   ↳ Skipping bonus calculation and moving to next employee...

=============================================
*** PERFORMANCE REVIEW SUMMARY ***
=============================================
Total Employees Processed: 5
High Performers (Rating ≥ 8): 2
Total Bonus Payout: $24280
Average Bonus per Employee: $4856

Note: Some employees were skipped due to missing performance data.

Processing completed at: 2024-01-20 14:30:25
*** SYSTEM PROCESSING COMPLETE ***

# 5. Concept Demonstration Summary
PL/SQL Concept	Demonstration in Code	Purpose & Best Practices
Collections		

  .VARRAY	dept_bonus_array for fixed bonus rates	Fixed-size collection, maintains order
  .Associative Array	dept_map for department name mapping	Key-value pairs, efficient lookup
  . Nested Table	emp_table_type for employee records	Dynamic sizing, bulk operations
  Records		
  .User-defined Record	emp_rec_type composite employee data	Groups related fields of different types
  .Table-based Record	Implicit in BULK COLLECT	Mirrors table structure
  .GOTO Statement	GOTO skip_bonus_calculation	Controlled jump for error handling
  .Bulk Processing	BULK COLLECT INTO v_employees	Efficient multi-row operations

# 6. Key Learning Points
Collections Demonstrated:

    VARRAY: Fixed-size, ordered collection for department bonus rates

    Associative Array: Key-value mapping for department codes

    Nested Table: Dynamic collection of employee records

Records Demonstrated:

    User-defined Records: Custom structure for employee performance data

    Composite Data Handling: Combining multiple data types in one variable

GOTO Statement Usage:

    Purpose: Controlled flow redirection for specific error conditions

    Best Practice: Used sparingly for clear, documented exceptional cases

    Alternative: Considered exception handlers vs GOTO for different scenarios
