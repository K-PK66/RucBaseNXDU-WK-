- Assignment Id: 3
- Assignment Name: homework
- Assignment Mode: Grading Mode
- Default Schema: sample_schema.sql
- Default dataset for data generation:  sample_data_small.sql    
- Default dataset(s) for evaluation:  sample_data_small.sql    
- Assignment end date: 2023-04-17 23:00:00.0
## List of Questions & Answers
**1. Query the student ID and grade of students who have taken CS-101(course_id) in Fall semester.**

Answer:

```sql
SELECT ID, grade
FROM takes
WHERE course_id='CS-101' and semester='Fall';
```

**2.Query the student ID and name of students who have taken CS-101(course_id) in Fall semester.

Answer:

```sql
SELECT takes.ID, student.name
FROM takes, student
WHERE course_id='CS-101' and semester='Fall' and takes.ID=student.ID;
```

**3.Query the student ID, name and grade of the students who have taken 'Database System Concepts'(course title), and the query results are sorted in descending order of grade.**

Answer:

```sql
SELECT takes.ID, student.name, takes.grade
FROM takes, student
WHERE takes.ID=student.ID and takes.course_id='CS-347'
ORDER BY grade DESC;
```
