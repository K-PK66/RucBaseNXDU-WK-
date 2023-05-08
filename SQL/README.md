- Assignment Id: 3
- Assignment Mode: Grading Mode
- Default Schema: sample_schema.sql
- Default dataset for data generation:  sample_data_small.sql    
- Default dataset(s) for evaluation:  sample_data_small.sql    
- Assignment end date: 2023-04-17 23:00:00.0
## List of Questions
**1. Query the student ID and grade of students who have taken CS-101(course_id) in Fall semester.**

**2. Query the student ID and name of students who have taken CS-101(course_id) in Fall semester.**

**3. Query the student ID, name and grade of the students who have taken 'Database System Concepts'(course title), and the query results are sorted in descending order of grade.**

**4. Query the student ID of students who have taken CS-101 or CS-190(course_id).**

**5. Query the name of students who have taken CS-101 or CS-190(course_id).**

**6. Query the student ID of students who do not take CS-101(course_id).**

**7. Query the course id of the prerequisite courses of CS-190(course_id).**

**8. Query the student ID of students who have taken all courses.**

**9. Query the student ID and name of students who have taken all courses which taken by student whose student id is '00128'.**

**10. Query the name, student ID and department name of students whose names begin with "S".**

**11. Query the name and student ID of the student whose second letter of name is "h".**

**12. Query the number of students who have taken course.**

**13. Query the maximum grade of students who have taken CS-101(course_id).**

**14. Query the total credits of elective courses taken by student whose student id(sno) is '00128'.**

**15. Query student ID of students who have taken 3 or more courses.**

**16. Query the student ID of students who have obtained 85 or more points in 2 or more courses and number of courses with 85 or more points for these students.**
## Answers:
**1. Query the student ID and grade of students who have taken CS-101(course_id) in Fall semester.**

```sql
SELECT ID, grade
FROM takes
WHERE course_id='CS-101' and semester='Fall';
```

**2. Query the student ID and name of students who have taken CS-101(course_id) in Fall semester.**

```sql
SELECT takes.ID, student.name
FROM takes, student
WHERE course_id='CS-101' and semester='Fall' and takes.ID=student.ID;
```

**3. Query the student ID, name and grade of the students who have taken 'Database System Concepts'(course title), and the query results are sorted in descending order of grade.**

```sql
SELECT takes.ID, student.name, takes.grade
FROM takes, student
WHERE takes.ID=student.ID and takes.course_id='CS-347'
ORDER BY grade DESC;
```

**4. Query the student ID of students who have taken CS-101 or CS-190(course_id).**

```sql
SELECT DISTINCT ID
FROM takes
WHERE course_id='CS-101' or course_id='CS-190';
```

**5. Query the name of students who have taken CS-101 or CS-190(course_id).**

```sql
SELECT name
FROM student
WHERE ID in (
  SELECT DISTINCT ID FROM takes WHERE course_id='CS-101' or course_id='CS-190'
);
```

**6. Query the student ID of students who do not take CS-101(course_id).**

```sql
SELECT DISTINCT ID
FROM student
WHERE NOT EXISTS(
  SELECT * FROM takes WHERE student.ID=ID and course_id='CS-101'
);
```

**7. Query the course id of the prerequisite courses of CS-190(course_id).**

```sql
SELECT prereq_id
FROM prereq
WHERE course_id='CS-190';
```

**8. Query the student ID of students who have taken all courses.**

```sql
SELECT ID
FROM student
WHERE NOT EXISTS(
  SELECT * FROM course WHERE NOT EXISTS(
	SELECT * FROM takes WHERE ID = student.ID and course_id=course.course_id
  )
);
```

**9. Query the student ID and name of students who have taken all courses which taken by student whose student id is '00128'.**

```sql
SELECT DISTINCT ID, name
FROM student
WHERE NOT EXISTS(
  SELECT * FROM takes t1 WHERE ID='00128' and NOT EXISTS(
	  SELECT * FROM takes t2 WHERE t1.course_id=t2.course_id and t2.ID=student.ID
	)
);
```

**10. Query the name, student ID and department name of students whose names begin with "S".**

```sql
SELECT name, ID, dept_name
FROM student
WHERE name LIKE 'S%';
```

**11. Query the name and student ID of the student whose second letter of name is "h".**

```sql
SELECT name, ID
FROM student
WHERE name LIKE '_h%';
```

**12. Query the number of students who have taken course.**

```sql
SELECT Count(DISTINCT ID)
FROM takes;
```

**13. Query the maximum grade of students who have taken CS-101(course_id).**

```sql
SELECT Max(grade)
FROM takes
WHERE course_id='CS-101';
```

**14. Query the total credits of elective courses taken by student whose student id(sno) is '00128'.**

```sql
SELECT SUM(course.credits)
FROM course, takes
WHERE takes.ID='00128' and takes.course_id=course.course_id;
```

**15. Query student ID of students who have taken 3 or more courses.**

```sql
SELECT ID FROM takes
GROUP BY ID HAVING Count(*)>=3;
```

**16. Query the student ID of students who have obtained 85 or more points in 2 or more courses and number of courses with 85 or more points for these students.**

```sql
SELECT ID, Count(*)
FROM takes
WHERE grade-85>=0
GROUP BY ID HAVING Count(*)>=2;
```
