---
layout: post
title: me vs. ChatGPT, Solving SQL Murder mystery, 
subtitle: Learning SQL, and seeing if AI can learn it too.
tags: [Confidence Intervals]
---


I am currenlty in business school and have been learning many new skills. One is SQL for data analysis, and another is finding out what AI tools are best to aide with tasks like Data Analysis. As an exercise for myself and an experiment into the SQL capabilities of ChatGPT, I decided to solve the interactive [SQL Murder Mystery](https://mystery.knightlab.com) myself, and to see how ChatGPT would fair. The goal was to brush up on my SQL abilities, and to see if ChatGPT could solve the mystery by itself. Since I am testing ChatGPT's SQL abilities, I made the rule for myself that I could not give ChatGPT SQL specific hints or instructions to start. This would defeat the purpose of using an AI to solve data analysis. If I knew what the SQL query should look like, I might as well type the SQL query myself.

Without further ado, let's start. 

### Getting Report Data

#### Me, a human. 

After taking a look at the schemam, I can see there is a `crime_scene_report` table. Let's take a look at that.

![Schema]({{"/img/murder_mystery_schema.png", | relative_url }})


Let's take a look at the `crime_scene_report` with the following query.

```SQL
SELECT * FROM crime_scene_report where city = 'SQL City' and date='20180115';
```

#### ChatGPT

After pasting the schema to the ChatGPT prompt, ChatGPT gave the following query

```SQL
SELECT *
FROM crime_scene_report
WHERE type = 'murder'
  AND date = '20180115'
  AND city = 'SQL City';
```

The two queries are fairly similar, so there is nothing to note. The result is below

```
Security footage shows that there were 2 witnesses. The first witness lives at the last house on "Northwestern Dr". The second witness, named Annabel, lives somewhere on "Franklin Ave".
````

### Finding the witnesses

Now we can find the witnesses,

#### Me. A human.

I want to find both witnesses in one go. To do this, I had to figure out how to write a correlated subquery, since the person with at the end of Northwestern Dr. has the highest address number, but I don't know what the highest house number is.

```sql
SELECT * 
FROM person 
WHERE 
(name like '%Annabel%' and address_street_name like '%Franklin Ave%') or 
(address_street_name like '%Northwestern Dr%' 
     and address_number = (
	   SELECT max(address_number) 
	   FROM  person WHERE address_street_name like '%Northwestern Dr%'))
```


#### ChatGPT.

I gave ChatGPT the result table and asked it to find the two witnesses. At first Chat GPT gives me two queries to run,

```SQL
SELECT *
FROM person
WHERE address_street_name = 'Northwestern Dr'
ORDER BY address_number DESC
LIMIT 1;
```
and 
```SQL
SELECT *
FROM person
WHERE name LIKE 'Annabel%'
  AND address_street_name = 'Franklin Ave';
```

These queries work, but it's no fun copy-pasting many queries, so I prompted ChatGPT to give me one query to find both witnesses, and this was the result.

```SQL
(
  SELECT *
  FROM person
  WHERE address_street_name = 'Northwestern Dr'
  ORDER BY address_number DESC
  LIMIT 1
)
UNION
(
  SELECT *
  FROM person
  WHERE name = 'Annabel'
    AND address_street_name = 'Franklin Ave'
);
```
Also, not really want I want, this is an innefficient query as well, since it needs to scan the person table twice. With another prompt, and a hint to use a correlated subquery, ChatGPT finally has a similar answer to mine
```SQL
SELECT *
FROM person
WHERE 
  (address_street_name = 'Northwestern Dr' AND address_number = (
    SELECT MAX(address_number)
    FROM person
    WHERE address_street_name = 'Northwestern Dr'
  ))
  OR 
  (name = 'Annabel' AND address_street_name = 'Franklin Ave');
```

##### Results

Both queries give us the following results.


id	| name	|license_id	|address_number	|address_street_name	|ssn
14887	|Morty Schapiro	|118009	|4919|	Northwestern Dr	|111564949
16371	|Annabel Miller	| 490173	| 103	| Franklin Ave	| 318771143


### Finding the transcripts

Now that we know the people, we can take a look at the transcripts. These queries are easy, since we have the person ids.

#### Me, a human

Now let's look and see if they have any interview transcripts

```sql
SELECT * FROM interview, person WHERE person.id in (14887, 16371) and person_id=id;
```

#### ChatGPT

```SQL
SELECT *
FROM interview
WHERE person_id IN (14887, 16371);
```

These SQL queries are basically the same, my sql query just also gives me the human readable person information from the person table.

#### Results

The interviews are as follows

```TEXT
I heard a gunshot and then saw a man run out. He had a "Get Fit Now Gym" bag. The membership number on the bag started with "48Z". Only gold members have those bags. The man got into a car with a plate that included "H42W".
```

```TEXT
I saw the murder happen, and I recognized the killer from my gym when I was working out last week on January the 9th.
```

We now have some interesting information about the killer.

1. the killer had a "Get Fit Now Gym" bag.
2. the membership number on the bag started with `48Z` and must be a gold member
3. the plate of the car include the symbols `H42W`
4. The killer was also at the gym on jan 9, 2018.
Now let's look at their check in/check out times, along with 


### Finding the Killer

#### Me, a human

Again, I want to write just one query, this is what I came up with.

```SQL
SELECT * FROM 
person,
get_fit_now_member,
drivers_license,
get_fit_now_check_in
WHERE 
person.id = get_fit_now_member.person_id and 
person.license_id = drivers_license.id and
get_fit_now_member.id like '48Z%' and
drivers_license.plate_number like '%H42W%' and 
get_fit_now_member.membership_status = 'gold' and
get_fit_now_check_in.membership_id = get_fit_now_member.id and
get_fit_now_check_in.check_in_date = 20180109;
```

#### ChatGPT

```SQL
SELECT 
    p.id AS person_id, 
    p.name, 
    dl.plate_number, 
    gfm.membership_id, 
    gfm.membership_status
FROM 
    get_fit_now_member gfm
JOIN 
    person p ON gfm.person_id = p.id
JOIN 
    drivers_license dl ON p.license_id = dl.id
WHERE 
    gfm.membership_id LIKE '48Z%' 
    AND gfm.membership_status = 'gold'
    AND dl.plate_number LIKE '%H42W%'
    AND EXISTS (
        SELECT 1 
        FROM get_fit_now_check_in gfc
        WHERE gfc.membership_id = gfm.id 
          AND gfc.check_in_date = 20180109
    );
```

Interesting the ChatGPT uses another correlated subquery to find if the killer checked in on the correct date. It looks like it could work, but after running the query in the playground I get the following error "Error: no such column: gfm.membership_id
", so I responded to ChatGPT with the message, and got the following query in response


```SQL
SELECT 
    p.id AS person_id, 
    p.name, 
    dl.plate_number, 
    gfm.membership_status
FROM 
    get_fit_now_check_in gfc
JOIN 
    get_fit_now_member gfm ON gfc.membership_id = gfm.id
JOIN 
    person p ON gfm.person_id = p.id
JOIN 
    drivers_license dl ON p.license_id = dl.id
WHERE 
    gfc.membership_id LIKE '48Z%' 
    AND gfm.membership_status = 'gold'
    AND dl.plate_number LIKE '%H42W%'
    AND gfc.check_in_date = 20180109;
````

This looks similar to the query I wrote. My query has join conditions as filter conditions, while ChatGPT explicity writes the join conditions. Depending on the Database, these two are basically equivalent. Let's look at the results.

#### Results

person_id	name	plate_number	membership_status
67318	Jeremy Bowers	0H42W2	gold

Seems like Jeremy Bowers is our guy. Let's look at his interview transcript to read his confession.

### Finding the Confession

#### Me, a human

```SQL
SELECT * FROM 
interview, person
WHERE person_id=id
and id = 67318;
```

#### ChatGPT

```SQL
SELECT *
FROM interview
WHERE person_id = 67318;
```


Both of the queries are almost similar.

#### Results

Jeremy's' confession reads as follows

```
I was hired by a woman with a lot of money. I don't know her name but I know she's around 5'5" (65") or 5'7" (67"). She has red hair and she drives a Tesla Model S. I know that she attended the SQL Symphony Concert 3 times in December 2017.
```

Seems like there is a bigger fish behind this. From the transcript, we know the real killer is

1. A women
2. With a lot of money
3. 5'5" or 5'7" or in between
4. Has red hair
5. Drives a tesla model S
6. attended the SQL symphony concert 3 times last year.

### Finding the real killer

#### Me, a human.

Let's try again a single query to find the real killer.

```SQL
SELECT * FROM
person,
drivers_license,
income, 
(SELECT 
	count(*) as count,
	person_id
 FROM
 	facebook_event_checkin
 WHERE event_name like '%SQL symphony concert%'
 group by 
 	person_id
 having count = 3
) as times_at_sql_symphony_concert
WHERE 
person.license_id = drivers_license.id and
income.ssn = person.ssn and 
times_at_sql_symphony_concert.person_id = person.id and
drivers_license.hair_color like '%red%' and 
drivers_license.gender like '%female%' and 
drivers_license.car_make like '%tesla%' and 
drivers_license.car_model like '%model s%';
```

#### ChatGPT

ChatGPT produces the following query
```SQL
SELECT p.id, p.name, p.height, p.hair_color, p.car_model
FROM person p
JOIN concert_attendance ca ON p.id = ca.person_id
WHERE (p.height = 65 OR p.height = 67)  -- Height conditions (in inches)
  AND p.hair_color = 'red'              -- Hair color condition
  AND p.car_model = 'Tesla Model S'    -- Car condition
  AND ca.event_name = 'SQL Symphony Concert'
  AND ca.event_date BETWEEN '2017-12-01' AND '2017-12-31'  -- Attending in December 2017
GROUP BY p.id, p.name, p.height, p.hair_color, p.car_model
HAVING COUNT(ca.event_date) = 3;  -- Attended exactly 3 times
```
But this errors out since that table concert_attendance doesn't exist, but `facebook_event_checkin` does. The query is pretty close though, it just needs to join a couple of more tables. I tell ChatGPT about the error, but not what tables it needs to join, otherwise I would just run the queries myself. The next query ChatGPT wants to try is below.
```SQL
SELECT *
FROM person
WHERE (height = 65 OR height = 67)  -- Height conditions
  AND hair_color = 'red'            -- Hair color condition
  AND car_model = 'Tesla Model S'; -- Car condition
```

This also doesn't make sense since height is not a column on person, but it is on driverse_license. Relaying this error to ChatGPT gives me the following query as another option.
```SQL
SELECT *
FROM person
WHERE name LIKE '%Annabel%' OR name LIKE '%Jeremy%';  -- Adjust as needed to filter relevant people by name
```
This is just way off. At this point I feel like I am spending more time herding ChatGPT to the correct answer than coming up with the answer myself. 


### Result,

Miranda Priestly! Looks like the fictional boss of Vogue magazine FROM the movie Devil Wears Prada is our killer. She has no interview, so I think we reached the end of the exercise.


### Conclusion

ChatGPT has some good SQL abilities, but it seems like if it needs to find a record given some complicated conditions, it fails. For example, the queries produced to find the two witnesses and the real killer produced a couple of issues. I'm sure you can explicitly say, "Join these tables to find ...", but I would expect ChatGPT to be able to do that.

