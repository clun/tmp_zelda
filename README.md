# Working with Zelda

## Data Model

### Create Tables

<p/>
<details>
<summary><b> Create Tables with a CQL Query(Cassandra Query Language)</b></summary>

```sql
/* ----------------------------------------- */
/* ----      TABLE   Objects          ------ */
/* ----------------------------------------- */

CREATE TABLE IF NOT EXISTS  objects (
	object_id UUID PRIMARY KEY,
    name TEXT,
    description TEXT,
    image TEXT,
    value_in_rupees INT,
    weight INT,
    type TEXT,
    stackable BOOLEAN,
	special MAP<TEXT,INT>);
 
/* ----------------------------------------- */
/* ----      TABLE  Characters        ------ */
/* ----------------------------------------- */

/** A character could be renamed. */
CREATE TABLE IF NOT EXISTS characters (
	character_id UUID PRIMARY KEY,
 	name TEXT,
	stamina INT,
	speed INT,
 	max_health INT,
 	current_health INT,
	weapon_slot UUID,
	shield_slot UUID);
	
/* Search character by name (if exist) with low cardinality. */
CREATE CUSTOM INDEX IF NOT EXISTS characters_name_idx ON characters (name) 
USING 'StorageAttachedIndex' 
WITH OPTIONS = {'case_sensitive': 'false', 'normalize': 'true', 'ascii': 'true'}; 

/* ----------------------------------------- */
/* ----       TABLE Inventory         ------ */
/* ----------------------------------------- */

CREATE TABLE IF NOT EXISTS inventory_by_character (
	character_id UUID,
	object_id UUID,
	object_name TEXT,
	weight INT,
	qty INT,
	PRIMARY KEY (character_id, object_name))
WITH CLUSTERING ORDER BY (object_name ASC);
```
</details>
<p/>

<p/>
<details>
<summary><b> Create Tables with GraphQL</b></summary>

<p>Please use endpoint ending with `api/graphql-schema`</p>
	
```yaml
mutation {
  table_objects: createTable(
    keyspaceName:"zelda"
    tableName:"objects"
    ifNotExists:true
    partitionKeys: [ 
      { name: "object_id", type: {basic: UUID} }
    ]
    values: [
      { name: "description", type: {basic: TEXT} },
      { name: "image", type: {basic: INT} },
      { name: "value_in_rupees", type: {basic: INT} },
      { name: "weight", type: {basic: INT} },
      { name: "type", type: {basic: TEXT} },
      { name: "stackable", type: {basic: BOOLEAN} },
      { name: "special", type: { 
          basic: MAP, 
          info: { subTypes: [ 
            { basic: TEXT }, 
            { basic: INT }
           ]
          }
      	}
      }
    ]
  )
  table_characters: createTable(
    keyspaceName: "zelda"
    tableName: "characters"
    partitionKeys: [{ name: "character_id", type: { basic: UUID } }]
    ifNotExists:true
    values: [
      { name: "name", type: {basic: TEXT} },
      { name: "stamina", type: {basic: INT} },
      { name: "speed", type: {basic: INT} },
      { name: "max_health", type: {basic: INT} },
      { name: "current_health", type: {basic: INT} },
      { name: "weapon_slot", type: {basic: UUID} },
      { name: "shield_slot", type: {basic: UUID} }
    ]
  )
}
```
</details>
<p/>




### DATASET

```sql
/* ----------------------------------------- */
/* ----             Objects           ------ */
/* ----------------------------------------- */

INSERT INTO objects (object_id, name, description, image, value_in_rupees, weight, type, stackable, special) VALUES (a7fdd361-4ac4-4f64-8ed0-ab9a951fc59f, 'Boko Shield', 'Found close to the blue Bokoblins, protects you from attack.', 'https://www.zeldadungeon.net/wiki/images/5/54/Boko-shield.png', 8, 1, 'shield', false, { 'durability': 7 });

INSERT INTO objects (object_id, name, description, image, value_in_rupees, weight, type, stackable, special) VALUES (03e81ae3-315a-4f17-8ea7-6119d7804145, 'Traveler’s Sword', 'Found in the Great Plateau, dropped by Bokoblins.', 'https://www.zeldadungeon.net/wiki/images/7/7b/Travelers-sword.png', 10, 1, 'sword', false, { 'durability': 7, 'damage': 5 });

INSERT INTO objects (object_id, name, description, image, value_in_rupees, weight, type, stackable, special) VALUES (959538f4-fafb-4113-8dea-80ce9a625816, 'Apple', 'Found on trees or barrels, restores half of a heart.', 'https://www.zeldadungeon.net/wiki/images/c/c3/Apple-botw.png', 2, 1, 'consumable', true, { 'health': 1});

INSERT INTO objects (object_id, name, description, image, value_in_rupees, weight, type, stackable, special) VALUES (638fd732-51fb-4230-8a87-2beb9e164691, 'Korok Seeds', 'Used to increase inventory squares.', 'https://www.zeldadungeon.net/wiki/images/0/01/Korok_Seed_-_HWAoC.png', 1, 0, 'item', true, { 'max_inventory_squares_plus': 1});

/* ----------------------------------------- */
/* ----             Characters        ------ */
/* ----------------------------------------- */

INSERT INTO characters(character_id, name, stamina, speed, 
			max_health, current_health,
 			weapon_slot, shield_slot) VALUES (
 			11111111-1111-1111-1111-111111111111, 'Cedrick', 10, 1, 6, 6, 03e81ae3-315a-4f17-8ea7-6119d7804145, a7fdd361-4ac4-4f64-8ed0-ab9a951fc59f);

INSERT INTO characters(character_id, name, stamina, speed, 
			max_health, current_health,
 			weapon_slot, shield_slot) VALUES (
 			22222222-2222-2222-2222-222222222222, 'Ania', 10, 1, 8, 10, 03e81ae3-315a-4f17-8ea7-6119d7804145, a7fdd361-4ac4-4f64-8ed0-ab9a951fc59f);

INSERT INTO characters(character_id, name, stamina, speed, 
			max_health, current_health,
 			weapon_slot, shield_slot) VALUES (
 			33333333-3333-3333-3333-333333333333, 'Aaron', 10, 1, 6, 6, 03e81ae3-315a-4f17-8ea7-6119d7804145, a7fdd361-4ac4-4f64-8ed0-ab9a951fc59f);	
			
/* ----------------------------------------- */
/* ----             Inventory         ------ */
/* ----------------------------------------- */

/* 10 Apples, weight is still 2 as stackable */
INSERT INTO inventory(character_id, object_id, object_name, weight, qty) 
VALUES(11111111-1111-1111-1111-111111111111, 959538f4-fafb-4113-8dea-80ce9a625816, 'Apple', 2, 10);
```

## Services

### C1 - Create Character

- **CQL**

```sql
INSERT INTO characters(
  character_id, name, stamina, 
  speed, max_health, current_health,
  weapon_slot, shield_slot) VALUES (
  11111111-1111-1111-1111-111111111111, 'Cedrick', 10, 1, 6, 6, 
  03e81ae3-315a-4f17-8ea7-6119d7804145, 
  a7fdd361-4ac4-4f64-8ed0-ab9a951fc59f);
```

- GraphQL

```yaml
```

### C2 - Find character by id

- **CQL**

```sql
select * from characters where character_id=11111111-1111-1111-1111-111111111111
```

### C3 - Is Character name in use ?

- **CQL**

```sql
select count(character_id) from characters where name='Ania';
select count(character_id) from characters where name='Link';
```

- **GraphQL**

```yaml
```

### C4 - Is Character name in use ?

```sql
select count(character_id) from characters where name='Ania';
select count(character_id) from characters where name='Link';
```

