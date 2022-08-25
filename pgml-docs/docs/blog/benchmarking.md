Benchmarking Rust vs Python Functions in Postgres
=================================================

Generating Random Embeddings
----------------------------
```sql
CREATE TABLE embeddings AS
SELECT id, ARRAY_AGG(rand) AS vector
FROM (
  SELECT row_number() over () % 10000 + 1 AS id, random()::REAL AS rand
  FROM generate_series(1, 1280000) AS t
) series
GROUP BY id
ORDER BY id;
```

6MB of data, tiny.

```sql
 \dt+ embeddings
                                      List of relations
 Schema |    Name    | Type  |  Owner  | Persistence | Access method |  Size   | Description
--------+------------+-------+---------+-------------+---------------+---------+-------------
 public | embeddings | table | montana | permanent   | heap          | 5728 kB |
(1 row)
```

```sql
SELECT * FROM embeddings LIMIT 1 OFFSET 9999;
```

```sql
  id   |



                                                                                                                                                                 vector





-------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 10000 | {0.4677334838785363,0.6199169805362921,0.008509639150592108,0.5498286015691214,0.07871558785325661,0.8348774190441794,0.6311723407102399,0.19212339727556937,0.4009401071299443,0.4433296885198992,0.08637908622108981,0.5807890489919068,0.9247415078083385,0.4776815048800849,0.16229492344792007,0.7429632683623062,0.9189767474432564,0.22078690362144826,0.9475800478605869,0.6344527428610185,0.7874705875445933,0.8748915523207792,0.8128351151340247,0.43367844829237256,0.6754609060773227,0.48057806313494567,0.642046553648246,0.08562096129077545,0.3260440099614961,0.08571130530089377,0.31403187698834145,0.09576577182071233,0.1866199475722432,0.16438371763564774,0.43388499644447975,0.5740485956767465,0.7653378212397719,0.28586062103630994,0.3144215335119078,0.5288006869835193,0.2375410997493681,0.033525530980060836,0.20762413239095068,0.41069415069575044,0.3022129308413959,0.17382547002415905,0.7860877724870114,0.9066872550982374,0.28842145371129746,0.0752162803059413,0.8744650809105714,0.9839697443058562,0.5117651115095931,0.37710793604282244,0.4309116449771899,0.18390815076356049,0.8808179638418814,0.9088098560322955,0.7785313247075045,0.16599110774638248,0.7535749992688388,0.8584762161519315,0.08682156532723084,0.5017746618594323,0.04389744780621996,0.1920512618593797,0.10311870976716264,0.7188275855378983,0.5779580934248578,0.3222146206923675,0.1340433111631718,0.5946766895470468,0.7006865865506633,0.027326614768700352,0.27694544535620835,0.7007521354822224,0.13221475656462545,0.6203735732862619,0.9013500233923004,0.19633274826884772,0.07432192430281148,0.36791314502301375,0.38040210402255425,0.8724913286624236,0.5668802145563667,0.1590196108369959,0.07231220620315426,0.446039965748529,0.09429986857151462,0.42422919616632626,0.551801621595363,0.10347534944282089,0.44197455654955675,0.30448538352920096,0.09154772706552095,0.29492408299103445,0.8027266901468728,0.07008422502389422,0.393629297185047,0.778087995468983,0.13325273497492063,0.7896196548287584,0.5709718167304381,0.9121894542825579,0.7605685231002361,0.24892880170222398,0.3787927931832691,0.2019166776677288,0.23845456554443345,0.12003730148279956,0.6960470692301932,0.09136904719054328,0.45390136478420473,0.3801046095890719,0.2568721352629417,0.008002420247127162,0.9835547272513203,0.9803693135197058,0.632033441878324,0.6565744425636844,0.7001610758326287,0.7650944453534443,0.46036971237822755,0.5630898606964969,0.6290127623700563,0.6405127942491085,0.930238255269618,0.868745834531925}
(2 rows)
```

Benchmarks
----------

### SQL
```sql
CREATE OR REPLACE FUNCTION sql_norm_l2(vector REAL[]) 
  	RETURNS REAL
	LANGUAGE sql
	LEAKPROOF
	IMMUTABLE
	STRICT
	PARALLEL SAFE
AS $$
  SELECT SQRT(SUM(squared.values))
  FROM (SELECT UNNEST(vector) * UNNEST(vector) AS values) AS squared;
$$;
```

```sql
SELECT sql_norm_l2(vector) FROM embeddings;
```
110ms

### PLPGSQL
```sql
CREATE OR REPLACE FUNCTION plpgsql_norm_l2(vector REAL[]) 
  	RETURNS REAL
	LANGUAGE plpgsql
	LEAKPROOF
	IMMUTABLE
	STRICT
	PARALLEL SAFE
AS $$
	BEGIN
		RETURN SQRT(SUM(squared.values))
		FROM (SELECT UNNEST(vector) * UNNEST(vector) AS values) AS squared;
	END
$$;
```

```sql
SELECT plpgsql_norm_l2(vector) FROM embeddings;
```
120 ms

### Python
```sql
CREATE OR REPLACE FUNCTION python_norm_l2(vector REAL[]) 
  	RETURNS REAL
	LANGUAGE plpython3u
	LEAKPROOF
	IMMUTABLE
	STRICT
	PARALLEL SAFE
AS $$
  import math
  return math.sqrt(sum([x * x for x in vector]))
$$;
```

```sql
SELECT python_norm_l2(vector) FROM embeddings;
```
-- 80ms

### Numpy
```sql
CREATE OR REPLACE FUNCTION numpy_norm_l2(vector REAL[]) 
  	RETURNS REAL
	LANGUAGE plpython3u
	LEAKPROOF
	IMMUTABLE
	STRICT
	PARALLEL SAFE
AS $$
  import numpy
  return numpy.linalg.norm(vector, ord=2)
$$;
```

```sql
SELECT numpy_norm_l2(vector) FROM embeddings;
```
-- 150ms

### Rust

```sql
SELECT pgx_norm_l2(vector) FROM embeddings;
```








Benchmarking SQL functions
--------------------------

Benchmarking PgPlSQL functions
-----------------------------

```sql
CREATE SCHEMA pgml;

---
--- Euclidean distance from the origin
---
CREATE OR REPLACE FUNCTION pgml.norm_l2(vector REAL[]) 
  	RETURNS REAL
	LANGUAGE plpgsql
	LEAKPROOF
	IMMUTABLE
	STRICT
	PARALLEL SAFE
AS $$
	BEGIN
		RETURN SQRT(SUM(squared.values))
		FROM (SELECT UNNEST(vector) * UNNEST(vector) AS values) AS squared;
	END
$$;

---
--- A projection of `a` onto `b`
---
CREATE OR REPLACE FUNCTION pgml.dot_product(a REAL[], b REAL[]) 
  	RETURNS REAL
	LANGUAGE plpgsql
	LEAKPROOF
	IMMUTABLE
	STRICT
	PARALLEL SAFE
AS $$
	BEGIN
		RETURN SUM(multiplied.values)
		FROM (SELECT UNNEST(a) * UNNEST(b) AS values) AS multiplied;
	END
$$;


---
--- The size of the angle between `a` and `b`
---
CREATE OR REPLACE FUNCTION pgml.cosine_similarity(a REAL[], b REAL[]) 
  	RETURNS REAL
	LANGUAGE plpgsql
	LEAKPROOF
	IMMUTABLE
	STRICT
	PARALLEL SAFE
AS $$
	BEGIN
		RETURN pgml.dot_product(a, b) / (pgml.norm_l2(a) * pgml.norm_l2(b));
	END
$$;
```

```sql