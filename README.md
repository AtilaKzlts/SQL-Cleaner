
# Data Cleaning With SQL

## Intrudciton
In this study, our aim is to clean our club membership table in our **PostgreSQL** database and make it ready for analysis.

*The raw version is attached below*;

| Full Name            | Age | Marital Status | Email                     | Phone         | Full Address                               | Job Title                        | Membership Date |
|----------------------|-----|---------------|---------------------------|--------------|--------------------------------------------|--------------------------------|----------------|
| Addie Lush          | 40  | Married       | alush0@shutterfly.com     | 254-389-8708 | 3226 Eastlawn Pass, Temple, Texas         | Assistant Professor            | 7/31/2013      |
| Rock Cradick        | 46  | Married       | rcradick1@newsvine.com    | 910-566-2007 | 4 Harbort Avenue, Fayetteville, North Carolina | Programmer III              | 5/27/2018      |
| Sydel Sharvell      | 46  | Divorced      | ssharvell2@amazon.co.jp   | 702-187-8715 | 4 School Place, Las Vegas, Nevada         | Budget/Accounting Analyst I    | 10/6/2017      |

## Objective

This SQL script does the following to clean and standardize the data in the **cleaned_clup** table:

#### 1. Parsing First and Last Name
- Clean up the `full_name` column and split it into `name` and `last_name`.  
- Combining multi-part surnames appropriately.  

#### 2. Edit Age and Marital Status
- Making illogical age values (`3 digits or empty`) `NULL`.  
- Assign empty `martial_status` values to `NULL`.  

#### 3. Formatting Contact Information
- Clean `email` data by converting it to lower case.  
- Edit `phone` numbers to contain only digits.  

#### 4. Splitting Address Data
- Split `full_address` data into `street_address`, `city` and `state`.  
- Update all address information in the appropriate format.  

#### 5. Standardizing Occupation Information
- Add `occupation` column and move `job_title` data there.  
- Update the levels (`I, II, III, IV`) to `“Level X”`.  


## Script
```sql
-- # I'm making a copy with the real data so it can't be damaged.
DROP TABLE IF EXISTS cleaned_clup

CREATE TABLE cleaned_clup AS  SELECT * FROM club_member_info ;  


-- # Let's start by clearing the full name.
UPDATE cleaned_clup
SET full_name = initcap(lower(regexp_replace(trim(both ' ' FROM full_name), '[^a-zA-Z\s]', '', 'g')));



-- # Adding new columns for first and last names.
ALTER TABLE cleaned_clup
ADD COLUMN name TEXT,
ADD COLUMN last_name TEXT;


-- # Assigning first names and last names for cases with 2-3 name parts; adding cases for last names with 2-3 parts. exp(María Pérez Rodríguez )
UPDATE cleaned_clup
SET
    name = split_part(trim(lower(full_name)), ' ', 1),
    last_name = CASE
        WHEN array_length(string_to_array(trim(lower(full_name)), ' '), 1) = 2 THEN split_part(trim(lower(full_name)), ' ', 2)
        WHEN array_length(string_to_array(trim(lower(full_name)), ' '), 1) = 3 THEN concat(split_part(trim(lower(full_name)), ' ', 2), ' ', split_part(trim(lower(full_name)), ' ', 3))
        WHEN array_length(string_to_array(trim(lower(full_name)), ' '), 1) = 4 THEN concat(split_part(trim(lower(full_name)), ' ', 2), ' ', split_part(trim(lower(full_name)), ' ', 3), ' ', split_part(trim(lower(full_name)), ' ', 4))
        ELSE ''
    END;


-- # Updating 'name' and 'last_name' columns using 'initcap'.
UPDATE cleaned_clup
SET
    name = initcap(name),
    last_name = initcap(last_name);



-- # Updating the age column; dealing with empty and 3-digit values.
UPDATE cleaned_clup
SET age = 
  CASE 
    WHEN LENGTH(age::text) = 0 THEN NULL
    WHEN LENGTH(age::text) = 3 THEN NULL
    ELSE age
  END;



-- # Assigning NULL to empty values.
UPDATE cleaned_clup 
SET martial_status =
    CASE 
        WHEN trim(martial_status) = '' THEN NULL
        ELSE trim(martial_status)
    END;



-- # Formatting and lowering email addresses.
UPDATE cleaned_clup
SET member_email = trim(lower(email));



-- # Formatting phone numbers.
UPDATE cleaned_clup
SET phone = CASE
    WHEN length(regexp_replace(phone, '[\s-]', '', 'g')) = 10 THEN regexp_replace(phone, '[\s-]', '', 'g')
    ELSE NULL
END;


-- # Adding new columns for address details.
ALTER TABLE cleaned_clup
ADD COLUMN street_address TEXT,
ADD COLUMN city TEXT,
ADD COLUMN state TEXT;


-- # Extracting and formatting street address, city, and state from full_address.
UPDATE cleaned_clup
SET
    street_address = trim(both ' ' from split_part(lower(full_address), ',', 1)),
    city = trim(both ' ' from split_part(lower(full_address), ',', 2)),
    state = trim(both ' ' from split_part(lower(full_address), ',', 3));
    
UPDATE cleaned_clup
SET
    street_address = initcap(street_address),
    city = initcap(city),
    state = initcap(state);



-- # Adding a column for job title.
ALTER TABLE cleaned_clup 
ADD COLUMN occupation TEXT;



--- # Formatting job titles and assigning levels.
UPDATE cleaned_clup
SET
    occupation = CASE
        WHEN trim(lower(job_title)) = '' THEN NULL
        ELSE 
            CASE
                WHEN array_length(string_to_array(trim(job_title), ' '), 1) > 1
                     AND lower(split_part(trim(job_title), ' ', array_length(string_to_array(trim(job_title), ' '), 1))) = 'i'
                    THEN replace(lower(trim(job_title)), ' i', ', level 1')

                WHEN array_length(string_to_array(trim(job_title), ' '), 1) > 1
                     AND lower(split_part(trim(job_title), ' ', array_length(string_to_array(trim(job_title), ' '), 1))) = 'ii'
                    THEN replace(lower(trim(job_title)), ' ii', ', level 2')

                WHEN array_length(string_to_array(trim(job_title), ' '), 1) > 1
                     AND lower(split_part(trim(job_title), ' ', array_length(string_to_array(trim(job_title), ' '), 1))) = 'iii'
                    THEN replace(lower(trim(job_title)), ' iii', ', level 3')

                WHEN array_length(string_to_array(trim(job_title), ' '), 1) > 1
                     AND lower(split_part(trim(job_title), ' ', array_length(string_to_array(trim(job_title), ' '), 1))) = 'iv'
                    THEN replace(lower(trim(job_title)), ' iv', ', level 4')

                ELSE trim(lower(job_title))
            END 
    END;

UPDATE cleaned_clup
SET
    occupation = initcap(occupation);


--- # Creating a new table with cleaned data.
CREATE TABLE new_cleaned (
    name VARCHAR(50),
    last_name VARCHAR(50),
    age INT,
    martial_status VARCHAR(50),
    occupation VARCHAR(100),
    email VARCHAR(100),
    phone VARCHAR(20),
    full_address VARCHAR(255),
    membership_date DATE,
    city VARCHAR(50),
    state VARCHAR(50)
);


--- # Inserting cleaned data into the new table.
INSERT INTO new_cleaned (name, last_name, age, martial_status, occupation, email, phone, full_address, membership_date, city, state)
SELECT name, last_name, age, martial_status, occupation, email, phone, full_address, membership_date, city, state
FROM cleaned_clup;
```

## Result
*Clean Table looks like*
| Name    | Last Name | Age | Martial Status | Occupation                   | Email                     | Phone      | Full Address                              | Membership Date | City         | State      |
|---------|----------|-----|---------------|------------------------------|---------------------------|------------|-------------------------------------------|----------------|--------------|------------|
| Morrie  | Overell  | 37  | Divorced      | Web Designer, Level 1       | moverellk@nydailynews.com | 5133796486 | 53061 Hoffman Park, Cincinnati, Ohio     | 2018-08-15     | Cincinnati   | Ohio       |
| Damaris | Dioniso  | 34  | Married       | Environmental Specialist     | ddionisol@utexas.edu      | 4155585275 | 5 Eagan Terrace, San Francisco, Kalifornia | 2012-02-21     | San Francisco | Kalifornia |
| Mort    | Paik     | 54  | Single        | Geologist, Level 4          | mpaik99@si.edu            | 3303372909 | 58 Petterle Point, Akron, Ohio           | 2019-01-24     | Akron        | Ohio       |
