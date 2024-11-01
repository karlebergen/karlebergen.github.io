---
layout: post
title: Solving SQL Murder mystery.
subtitle: Learning SQL, and seeing if AI can learn it too.
tags: [Confidence Intervals]
---


I decided to dive into the thrilling interactive experience where I become the detective in a gripping murder case! I decided to play [SQL Murder Mystery](https://mystery.knightlab.com) to learn and practice SQL skills while unraveling a captivating story. I thought it might also be fun to document my learnings!

Looking at the schema I can see there is a `crime_scene_report` table. Let's take a look at that.

```SQL
SELECT * FROM crime_scene_report where city = 'SQL City' and date='20180115';
```

This gives us only one murder on that day. With the following description.

```
Security footage shows that there were 2 witnesses. The first witness lives at the last house on "Northwestern Dr". The second witness, named Annabel, lives somewhere on "Franklin Ave".
```

Let's look up information on these witnesses. The maximum address an northwestern street is an interesting curveball. But we can solve that with a correlated subqueries.

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

That gives us the following results


id	| name	|license_id	|address_number	|address_street_name	|ssn
14887	|Morty Schapiro	|118009	|4919|	Northwestern Dr	|111564949
16371	|Annabel Miller	| 490173	| 103	| Franklin Ave	| 318771143


Now let's look and see if they have any interview transcripts

```sql
SELECT * FROM interview, person WHERE person.id in (14887, 16371) and person_id=id;
```

We join the tables so we can also get the persons name. The interviews give us some interesting information, 

1. the killer had a "Get Fit Now Gym" bag.
2. the membership number on the bag started with `48Z` and must be a gold member
3. the plate of the car include the symbols `H42W`
4. The killer was also at the gym on jan 9, 2018.
Now let's look at their check in/check out times, along with 


Let's see if we can find the killer in one go with the following query

```SQL
SELECT * FROM 
person,
get_fit_now_member,
drivers_license
WHERE 
person.id = get_fit_now_member.person_id and 
person.license_id = drivers_license.id and
get_fit_now_member.id like '48Z%' and
drivers_license.plate_number like '%H42W%' and 
get_fit_now_member.membership_status = 'gold';
```

That returns a person with the name "Jeremy Bowers". Let's also see if they were checked into the gym on Jan 9, 2018.


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

Looks like we found him! Jeremy Bowers is our guy. But it turns out there is more, we need to query his interview transcript to find out more information.


```SQL
SELECT * FROM 
interview, person
WHERE person_id=id
and id = 67318;
```

From the transcript, we know the real killer is

1. A women
2. With a lot of money
3. 5'5" or 5'7" or in between
4. Has red hair
5. Drives a tesla model S
6. attended the SQL symphony concert 3 times last year.

Let's try to write a query to find this person!

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

and that leads us to Miranda Priestly! Looks like the fictional boss of Vogue magazine FROM the movie Devil Wears Prada is our killer. 

### Can AI Do this?

-- TODO


