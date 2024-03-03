# Advance topics

__TODO__ : add clear and small examples in relevance to complex queries.

## import demo database inside postgres container

We are going to create a docker compose file to move the demo database into the image and
then open it
- download demo database from : [https://edu.postgrespro.com/demo-small-en.zip](https://edu.postgrespro.com/demo-small-en.zip)
- go to terminal
    - `vim .env`
        - paste the following and enter the valid paths:
      ```text
      DEMO_DATABASE_PATH="<path to demo file>/demo-small-en-20170815.sql"
      DATABASE_BACKUP="<path to db backup"
      #POSTGRES_DB="flight"
      POSTGRES_USER="flight"
      POSTGRES_PASSWORD="flight"
      ```

    - `vim docker-compose.yml`
        - paste the following
  ```yaml
    version: '3'
    services:
    postgres:
    image: postgres:latest
    restart: always  
    container_name: postgresintro
    environment:
    POSTGRES_USER: ${POSTGRES_USER}
    POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}  
    ports:
    - "5432:5432"
    volumes:
      - ${DEMO_DATABASE_PATH}:/demo-small-en-20170815.sql
      - ${DATABASE_BACKUP}:/var/lib/postgresql/data
  ```

    - run : `docker composer up -d`
        -  `docker exec -it postgresintro psql -d flight -U flight`
        -  `\i demo-small-en-20170815.sql`
        - db will automatically change to demo

    - to change to database demo next time,
        - `docker exec -it postgresintro psql -d demo -U flight`
        - OR : `\c demo` inside psql

___

## DEMO database

### Schema
- The demo database imported contains statistics on all the flights of an imaginary airline company of one month time-period
- In postgres, schema is a named collection of database objects, including tables, views, functions, i.e, a collection
  logical structures of data.
- The main entity in our demo database's schema is a __booking__(mapped to bookings table).
    - Each booking includes several passengers with a __separate__ ticket issued for each passenger(tickets).
        - we assume each passenger is unique , there is no unique-id issued for each passenger as a person
    - Each ticket always contain one or more flight segments(ticker_flights).
        - if there are no direct flights with departure to destination, and it's not round trip ticket then a ticket can
          have multiple flight segments included.
            - it's assumed all tickets in a single booking has the same flight segment. Even though schema does not contain
              any such restrictions.
    - Each flight('flight') goes from one airport('airports') to another .
        - Flights with the same flight number have the same points of departure and destination but different departure
          dates.
    - At flight check-in, each passenger is issued a boarding pass (boarding_passes), where the seat number is specified.
        - The flight/seat combination must be unique

> you can use \d+ command to understand tables within terminal too.
>  \d+ bookings
> \d+ aircrafts

### Bookings
- To fly with our airline, passengers book the required tickets in advance (book_date, which must be within one month
  before the flight). The booking is identified by its number (book_ref, a six-position combination of letters and digits).
- The total_amount field stores the total price of all tickets included into the booking, for all passengers.

### Tickets

- A ticket has a unique number (ticket_no), which consists of 13 digits.
    - Ticker contains passenger_id, passenger_name and their contact_data

### Flight Segments

- A flight segment connects a ticket with a flight, identified by a number
- Each flight segment has its price (amount) and travel class(fare_conditions).

### Flights

- natural composite key of the flights table consists of the flight number (flight_no) and the date of the departure
  (scheduled_departure). To make foreign keys that refer to this table a bit shorter, a surrogate key flight_id is used as
  the primary key.

- A flight always connects two points: departure_airport and arrival_airport.
    - There's no entity like "connecting flight" , in such cases . Ticket contains all flight segments.
- Each flight has time of scheduled_departure and scheduled_arrival, actual_departure and actual_arrival times may differ.

#### Flight status
- Scheduled :The flight can be booked, in advance one month before departure.
- On Time : The flight is open for check-in (twenty-four hours before the scheduled departure) and is not delayed.
- Delayed : open for check-in but delayed
- Departed : airborne
- Arrived
- Cancelled

### Airports

- identified by a three-letter airport_code and has an airport_name.
- a city name is simply an airport attribute, which is required to identify all the airports of the same city.
    - The table also includes coordinates (longitude and latitude) and the timezone.
### Boarding Passes
- the boarding pass is identified by the combination of ticket and flight numbers.
- Boarding pass numbers (boarding_no) are assigned sequentially and has seat number (seat_no).

### Aircraft
- To identify an aircraft model, a three-digit aircraft_code
- also includes the name of the air-craft model and the maximum flying distance, in kilometers(range).

### Seats
-  define the cabin configuration of each aircraft model.
- Each seat has a number (seat_no) and an assigned travel class (fare_conditions): Economy, Comfort, or Business.

### Flights View

__NOTE__: View is a query stored in postgres database server.

- There is a flights_v view built over the flights table.
- use \d+ flights_v

### The “now” Function
- The demo database contains a snapshot of data
- The snapshot time is saved in the `bookings.now` function.
    -  use this function in demo queries for cases that would typically require calling the `now` function.
    - the return value of this function determines the version of the demo database: `SELECT bookings.now();`

#### Simple Queries

>Problem: Who traveled from Moscow (SVO) to Novosibirsk
(OVB) on seat 1A the day before yesterday, and when was the
ticket booked?

> Solution : use bookings.now()
```SQL
SELECT t.passenger_name,
       bb.book_date
FROM bookings b
      JOIN tickets t
        ON t.book_ref = b.book_ref
      JOIN boarding_passes bp
        ON bp.ticket_no = t.ticket_no
      JOIN flights f
        ON f.flight_id = bp.flight_id
WHERE f.departure_airport = 'SVO'
AND f.arrival_airport = 'OVB'
AND f.scheduled_departure::date =
    bookings.now()::date - INTERVAL '2 day'
AND bp.seat_no = '1A';
```
__NOTE__: '::' is used to data from one type to another :


> Problem: How many seats remained free on flight PG0404 yesterday?
> Solution: Let's try to use the NOT EXISTS expression for seats without boarding-pass.
```SQL
SELECT count(*)
FROM flights f
     JOIN seats s
     ON s.aircraft_code = f.aircraft_code
WHERE f.flight_no = 'PG0404'
AND f.scheduled_departure::date =
    bookings.now()::date - INTERVAL '1 day'
AND NOT EXISTS (
                SELECT NULL
                FROM boarding_passes bp
                WHERE bp.flight_id = f.flight_id
                AND bp.seat_no = s.seat_no
               );
```

>Problem: Which flights had the longest delays? Print the list of ten “leaders.”
> Solution: The query needs to include only those flights that have already departed.
```SQL
SELECT f.flight_no,
f.scheduled_departure,
f.actual_departure,
f.actual_departure - f.scheduled_departure
AS delay
FROM flights f
WHERE f.actual_departure IS NOT NULL
ORDER BY f.actual_departure - f.scheduled_departure
DESC
LIMIT 10;
```

#### Aggregate Functions
>Problem: What is the shortest flight duration for each
possible flight from Moscow to St. Petersburg, and how many
times was the flight delayed for more than an hour?

> Solution: flights_v view instead of dealing with table joins,  take into account only those flights that have
already arrived.
```SQL
SELECT  f.flight_no,
        f.scheduled_duration,
        min(f.actual_duration),
        max(f.actual_duration),
        sum(CASE WHEN f.actual_departure > f.scheduled_departure + INTERVAL '1 hour'
                      THEN 1 ELSE 0
            END) delays
FROM flights_v f
WHERE f.departure_city = 'Moscow'
      AND f.arrival_city = 'St. Petersburg'
      AND f.status = 'Arrived'
GROUP BY f.flight_no,
         f.scheduled_duration;
```

> Problem: Find the most disciplined passengers who checked
in first for all their flights. Take into account only those
passengers who took at least two flights.

> Solution : boarding pass numbers are issued in the check-in order.
```SQL
SELECT  t.passenger_name,
        t.ticket_no
FROM    tickets t
JOIN  boarding_passes bp
        ON bp.ticket_no = t.ticket_no
GROUP BY t.passenger_name,
      t.ticket_no
HAVING max(bp.boarding_no) = 1
AND count(*) > 1;
```

> Problem: How many passengers can be included into a single booking?
> Solution: count the number of passengers in each booking and then find the number of bookings for each
number of passengers.
```SQL
SELECT  tt.cnt,
        count(*)
FROM  (
        SELECT  t.book_ref,
                count(*) cnt
        FROM tickets t
        GROUP BY t.book_ref
      ) tt
GROUP BY tt.cnt
ORDER BY tt.cnt;
```

#### Window Functions
>Problem: For each ticket, display all the included flight segments, together with connection time. Limit the result
to the tickets booked a week ago.
>Solution: Use window functions to avoid accessing the same data twice.
```SQL
SELECT  tf.ticket_no,
        f.departure_airport,
        f.arrival_airport,
        f.scheduled_arrival,
        lead(f.scheduled_departure) OVER w AS next_departure,
        lead(f.scheduled_departure) OVER w - f.scheduled_arrival AS gap
FROM    bookings b
        JOIN  tickets t
          ON t.book_ref = b.book_ref
        JOIN  ticket_flights tf
          ON tf.ticket_no = t.ticket_no
        JOIN  flights f
          ON tf.flight_id = f.flight_id
WHERE b.book_date =
      bookings.now()::date - INTERVAL '7 day'
WINDOW w AS ( PARTITION BY tf.ticket_no
              ORDER BY f.scheduled_departure);
```

#### Arrays
> Problem. Find the round-trip tickets in which the outbound route differs from the inbound one.
> A round-trip ticket where the outbound route differs from the inbound route is typically referred to as a "multi-city"
> or "multi-destination" ticket. With this type of ticket, travelers can fly from one location to another
> (the outbound route), and then return from a different location (the inbound route).

> Solution:
```SQL
SELECT  r1.departure_airport,
        r1.arrival_airport,
        r1.days_of_week dow,
        r2.days_of_week dow_back
FROM    routes r1
JOIN    routes r2
          ON r1.arrival_airport = r2.departure_airport
        AND r1.departure_airport = r2.arrival_airport
WHERE   NOT (r1.days_of_week && r2.days_of_week);
```

#### Recursive Queries
[Guide](https://habr.com/en/companies/postgrespro/articles/490228/)
```SQL
WITH RECURSIVE t(n,factorial) AS (
  VALUES (0,1)
  UNION ALL
  SELECT t.n+1, t.factorial*(t.n+1) FROM t WHERE t.n < 5
)
SELECT * FROM t;
```

>Problem. How can you get from Ust-Kut (UKX) to Neryungri (CNN) with the minimal number of connections? What will
the flight time be?
>Solution: find the shortest path in the graph.

```SQL
WITH RECURSIVE p(
  last_arrival,
  destination,
  hops,
  flights,
  flight_time,
  found
) AS (
  SELECT  a_from.airport_code,
          a_to.airport_code,
          array[a_from.airport_code],
          array[]::char(6)[],
          interval '0',
          a_from.airport_code = a_to.airport_code
FROM  airports a_from,
      airports a_to
WHERE a_from.airport_code = 'UKX'
AND a_to.airport_code = 'CNN'
UNION ALL
SELECT  r.arrival_airport,
        p.destination,
        (p.hops || r.arrival_airport)::char(3)[],
        (p.flights || r.flight_no)::char(6)[],
        p.flight_time + r.duration,
        bool_or(r.arrival_airport = p.destination)
          OVER ()
FROM p
      JOIN routes r
      ON r.departure_airport = p.last_arrival
WHERE NOT r.arrival_airport = ANY(p.hops)
AND NOT p.found
)
SELECT hops,
        flights,
        flight_time
FROM p
WHERE p.last_arrival = p.destination;
```

> Problem. What is the maximum number of connections that can be required to get from any airport to any other airport?
> Solution: We can take the previous query as the basis for the solution. However, the first iteration must now contain
> all the possible airport pairs, not just a single pair: each airport must be connected to all the other airports.
> For all these pairs of airports we first find the shortest path, and then select the longest of them.

```SQL
WITH  RECURSIVE p(
      departure,
      last_arrival,
      destination,
      hops,
      found
) AS (
SELECT  a_from.airport_code,
        a_from.airport_code,
        a_to.airport_code,
        array[a_from.airport_code],
        a_from.airport_code = a_to.airport_code
FROM    airports a_from,
        airports a_to
UNION ALL
SELECT  p.departure,
        r.arrival_airport,
        p.destination,
        (p.hops || r.arrival_airport)::char(3)[],
        bool_or(r.arrival_airport = p.destination)
          OVER (PARTITION BY p.departure, p.destination)
FROM p JOIN routes r
        ON r.departure_airport = p.last_arrival
WHERE NOT r.arrival_airport = ANY(p.hops)
AND NOT p.found
)
SELECT max(cardinality(hops)-1)
FROM p
WHERE p.last_arrival = p.destination;
```