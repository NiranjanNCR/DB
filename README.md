--Question 1

---Define a data structure to represent the given dataset. This would be a schema in SQL
  
CREATE TABLE EmployeeHierarchy (
    employee_name VARCHAR(50) PRIMARY KEY,
    manager_name VARCHAR(50) REFERENCES EmployeeHierarchy(employee_name)
);

insert into EmployeeHierarchy values('mohan', 'syam');
insert into EmployeeHierarchy values('abdul', 'syam');
insert into EmployeeHierarchy values('ram', 'syam');
insert into EmployeeHierarchy values('syam', 'sandeep');
insert into EmployeeHierarchy values('peter', 'sandeep');
insert into EmployeeHierarchy values('vikram', 'peter');
insert into EmployeeHierarchy values('baljeet', 'peter');
insert into EmployeeHierarchy values('sunder', 'baljeet');

select* from EmployeeHierarchy;


/*Question 2

Implement the following queries on the data representation chosen in Q1

List the direct reports of a manager
List the full suborganization that reports to a manager
Show the complete management chain of an employee
Check for loops in the organization*/

--1) List the direct reports of a manager

SELECT employee_name
FROM EmployeeHierarchy
WHERE manager_name = 'syam';



--2) List the full suborganization that reports to a manager:

WITH RECURSIVE Suborganization AS (
    SELECT employee_name, manager_name
    FROM EmployeeHierarchy
    WHERE manager_name = 'syam'
    UNION ALL
    SELECT e.employee_name, e.manager_name
    FROM EmployeeHierarchy e
    INNER JOIN Suborganization s ON e.manager_name = s.employee_name
)
SELECT * FROM Suborganization;


--3) Show the complete management chain of an employee:

WITH RECURSIVE ManagementChain AS (
    SELECT employee_name, manager_name
    FROM EmployeeHierarchy
    WHERE employee_name = 'peter'
    UNION ALL
    SELECT e.employee_name, e.manager_name
    FROM EmployeeHierarchy e
    INNER JOIN ManagementChain m ON e.employee_name = m.manager_name
)
SELECT * FROM ManagementChain;



--4) Check for loops in the organization:

WITH RECURSIVE OrganizationPaths AS (
    SELECT employee_name, manager_name, ARRAY[employee_name] AS path
    FROM EmployeeHierarchy
    UNION ALL
    SELECT e.employee_name, e.manager_name, op.path || e.employee_name
    FROM EmployeeHierarchy e
    INNER JOIN OrganizationPaths op ON e.manager_name = op.employee_name
)
SELECT DISTINCT employee_name, manager_name, path
FROM OrganizationPaths
WHERE array_length(path, 1) > 1
ORDER BY path;



/*Question 3

Implement the following updates on the representation chosen in Q1. The updates would be transactions in SQL / methods in C++

Add a given employee as a direct report of a given manager. Validate that
given employee does not exist
given manager does exist
the entry will not create a loop (use the query of Q1-D for this purpose)

Remove a given employee. Validate that
Removal will not create orphans (employees whose management chain is broken/disconnected)

Move an employee from one manager to another, along with the employee's suborganization if any*/


-- Add a given employee as a direct report of a given manager:

BEGIN;
-- if the given employee does not exist
IF NOT EXISTS (SELECT 1 FROM EmployeeHierarchy WHERE employee_name = 'anand') THEN
    
    IF EXISTS (SELECT 1 FROM EmployeeHierarchy WHERE employee_name = 'syam') THEN
        
        WITH RECURSIVE OrganizationPaths AS (
            SELECT employee_name, manager_name, ARRAY[employee_name] AS path
            FROM EmployeeHierarchy
            UNION ALL
            SELECT e.employee_name, e.manager_name, op.path || e.employee_name
            FROM EmployeeHierarchy e
            INNER JOIN OrganizationPaths op ON e.manager_name = op.employee_name
        )
        SELECT path FROM OrganizationPaths WHERE path @> ARRAY['manager_name_here']::VARCHAR[] LIMIT 1 INTO @manager_path;

        IF @manager_path IS NULL THEN
           
            INSERT INTO EmployeeHierarchy (employee_name, manager_name) VALUES ('anand', 'syam');
            COMMIT;
        ELSE
           
            ROLLBACK;
        END IF;
    ELSE
        
        ROLLBACK;
    END IF;
ELSE

    ROLLBACK;
END IF;



--Remove a given employee:

BEGIN;

WITH Orphans AS (
    SELECT DISTINCT e.employee_name
    FROM EmployeeHierarchy e
    LEFT JOIN EmployeeHierarchy sub ON e.employee_name = sub.manager_name
    WHERE sub.employee_name IS NULL
)
SELECT employee_name FROM Orphans WHERE employee_name = 'anand' LIMIT 1 INTO @orphan_check;

IF @orphan_check IS NULL THEN
    
    DELETE FROM EmployeeHierarchy WHERE employee_name = 'anand';
    COMMIT;
ELSE
   
    ROLLBACK;
END IF;




--Move an employee from one manager to another:

BEGIN;
-- Check if the source employee exists and if the destination manager exists
IF EXISTS (SELECT 1 FROM EmployeeHierarchy WHERE employee_name = 'peter') AND
   EXISTS (SELECT 1 FROM EmployeeHierarchy WHERE employee_name = 'sunder') THEN
 
    WITH RECURSIVE OrganizationPaths AS (
        SELECT employee_name, manager_name, ARRAY[employee_name] AS path
        FROM EmployeeHierarchy
        UNION ALL
        SELECT e.employee_name, e.manager_name, op.path || e.employee_name
        FROM EmployeeHierarchy e
        INNER JOIN OrganizationPaths op ON e.manager_name = op.employee_name
    )
    SELECT path FROM OrganizationPaths WHERE path @> ARRAY['sunder']::VARCHAR[] LIMIT 1 INTO @manager_path;

    IF @manager_path IS NULL THEN
        -- It will not create a loop, so move the employee
        UPDATE EmployeeHierarchy SET manager_name = 'syam' WHERE employee_name = 'sandeep';
        COMMIT;
    ELSE
       
        ROLLBACK;
    END IF;
ELSE
    
    ROLLBACK;
END IF;
