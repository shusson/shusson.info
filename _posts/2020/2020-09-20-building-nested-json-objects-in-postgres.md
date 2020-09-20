# Building nested json objects in postgres

__20/09/2020__

If you are using a relational data model, it's often the application layer that will transform normalized data from the db into a hierarchical object. For example many ORMs will support something like `formRepository.find({ relations: ["section", "section.question"] });` which will do something like [this](https://github.com/typeorm/typeorm/blob/c714867d3d0c43ccbb7ca8fb3ce969207e4d5c04/src/query-builder/SelectQueryBuilder.ts#L1926) behind the scenes. Moving the normalized data between the db and the application layer is redundant in space and time. With [json functions](https://www.postgresql.org/docs/12/functions-json.html), postgres can do the transformations instead.

Basic model:
```sql
DROP TABLE IF EXISTS question;
DROP TABLE IF EXISTS section;
DROP TABLE IF EXISTS form;


CREATE TABLE form (
    id serial primary key,
    description text
);

CREATE TABLE section (
    id serial primary key,
    name text,
    form_id serial references form(id)
);

CREATE TABLE question (
    id serial primary key,
    name text,
    section_id serial references section(id)
);
```

Generate some data:
```sql
INSERT INTO
    form (description)
VALUES
    (
        md5(random()::text)
    ),
    (
        md5(random()::text)
    );

INSERT INTO
    section (name, form_id)
SELECT
    md5(random()::text),
    1
FROM
    generate_series(1, 10) s(i);


INSERT INTO
    question (name, section_id)
SELECT
    md5(random()::text),
    floor(i)
FROM
    generate_series(1, 10, 0.2) s(i);
```

Fetch the normalized representation of a form:
```sql
SELECT
	form.id AS form_id,
	section.id AS section_id,
	question.id AS question_id
FROM form
LEFT JOIN section ON section.form_id = form.id
LEFT JOIN question ON question.section_id = form.id
WHERE form.id = 1;

form_id|section_id|question_id|
-------|----------|-----------|
      1|         1|          1|
      1|         1|          2|
      1|         1|          3|
      1|         1|          4|
...
```

Fetch the JSON objects of a form using nested sub-queries:
```sql
SELECT row_to_json(forms)
FROM (
    SELECT
    	form.*,
        (
        	SELECT jsonb_agg(nested_section)
        	FROM (
	        	SELECT
		     		section.id,
		     		section.name,
		     		(
		     			SELECT json_agg(nested_question)
		     			FROM (
		     				SELECT
		     				question.id,
		     				question.name
			     			FROM question
			     			where question.section_id = section.id
		     			) AS nested_question
		     		) AS questions
		        FROM section
		        WHERE section.form_id = form.id
        	) AS nested_section
        ) AS sections
    FROM form
) AS forms;
```

Each row will look like:
```json
{
    "id": 1,
    "description": "53f6420b9ba0c269a9eb611a47480536",
    "sections": [
        {
            "id": 1,
            "name": "3d670fbce46e67b67a42eb743a700529",
            "questions": [
                {
                    "id": 1,
                    "name": "ee4918cd0b77b3fe2b7a4f5a8fd1c163"
                },
                {
                    "id": 2,
                    "name": "a1337f89c3cb2c369ef959741d5aed33"
                },
                {
                    "id": 3,
                    "name": "c0db64d35a0253561514ee929d813101"
                }
                ...
            ]
        },
    ...
```

Alternatively you can use Common Table Expressions:

```sql
WITH questions as (
    SELECT
      question.*
    FROM question
    GROUP BY question.id
    order by question.id
), sections AS (
    SELECT
      section.*,
      json_agg(questions) as questions
    FROM section
    LEFT JOIN questions ON questions.section_id = section.id
    GROUP BY section.id
    order by section.id
), forms AS (
    SELECT
      form.*,
      json_agg(sections) as sections
    FROM form
    LEFT JOIN sections ON sections.form_id = form.id
    group by form.id
    order by form.id
)
SELECT row_to_json(forms)
FROM forms;
```

In a following blog post I plan to investigate the performance these json functions.
