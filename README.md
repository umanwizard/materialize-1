This is a demo of Materialize using the Custom Labels feature from Parca/Polar Signals.

The demo fixes `mz_ore::task` to propagate labels to new tasks, and also initially sets a label
for the current username when the coordinator is processing a command or message.

The library then ensures that the labels are always set (and thus visible to the profiler) whenever any future transitively spawned from that point is running.

It also adds a 100ms CPU spin hidden in join optimization, so anyone executing a join will waste CPU in environmentd.

To run it, build Materialize and then run this script:

``` bash
#!/usr/bin/env bash

trap 'kill 0' SIGINT

echo '
DROP TABLE IF EXISTS t1;
DROP TABLE IF EXISTS t2;

CREATE TABLE t1 (x int);
CREATE TABLE t2 (x int);

DROP ROLE IF EXISTS alice;
DROP ROLE IF EXISTS bob;

CREATE ROLE alice;
CREATE ROLE bob;

GRANT ALL ON t1,t2 TO alice, bob;' | psql -h localhost -p 6875 materialize materialize

(while true; do
     echo 'SELECT * FROM t1 INNER JOIN t2 USING (x);' | psql -h localhost -p 6875 materialize alice >& /dev/null
 done) &

(while true; do
     echo 'SELECT sum(x) FROM t1;' | psql -h localhost -p 6875 materialize bob >& /dev/null
 done) &

wait
```


Then, in Parca or Polar Signals, group by the "username" label, and notice that stacks under `"alice"` are spending time in join planning.
