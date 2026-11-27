Suggestion: The nightly scans' pods ask the scheduler for 512Mi and use 1.8 to 3.2 GiB, the control pods for 50m CPU and use three times it, the managers for 250m and use a sixth: set LOOPER_REQUESTS_scan, _control and _manager from the recap's measurements while they still run in the cluster
Type: Operability
Summary: Tonight's recap (requests, looper#165): the 24 scan pods requested 512Mi of memory and used p50 1816Mi, p90 2333Mi, max 3204Mi (the second night over the threshold; 2026-11-26: p90 2715Mi, max 2766Mi over 38 pods), and 150m CPU against a p90 of 170m; the 89 control pods requested 50m CPU and used p90 159m, max 225m, the seventh night running; the 82 manager pods requested 250m CPU and used p90 38m, 'the pool full on paper and idle in fact'. The scheduler packs a c4a-standard-2 node by what is requested, so four nightly scans that fit on paper want 9 GiB of its 6, and the nightly scan, the job that holds the manager's token and files the night's bugs and proposals, is the pod the kubelet evicts first when memory runs short, since it is the furthest over its request. Set looper-dispatch's LOOPER_REQUESTS_scan memory to 2560Mi (limit 3.5Gi), LOOPER_REQUESTS_control CPU to 200m and LOOPER_REQUESTS_manager CPU to 50m in ci/tacplus-dev.gitlab-ci.yml, the CI file the looper project reads, so the node's room is counted right until the scans leave the cluster with the builds
Why: The migration keeps the in-cluster control runner for the kicker, the managers and the scans until its steps 4 to 6 (decision 2026-11-24), weeks away, and Jody asked that 'recap should let us know if we need to adjust': it has, seven nights for control and two for the scans. Three values in one file, landed through the looper's manager. Also serves Resiliency: an evicted scan leaves a repository without its night's filing

## Evidence
- https://gitlab.com/tacplus-dev/looper/-/jobs/18423975993 (recap 2026-11-27, requests: scan pods (24) cpu requested 150m, used p50 111m, p90 170m, max 240m; memory requested 512Mi, used p50 1816Mi, p90 2333Mi, max 3204Mi; control pods (89) cpu requested 50m, used p50 115m, p90 159m, max 225m; manager pods (82) cpu requested 250m, used p50 17m, p90 38m, max 73m; recap-classes: requests/scan seen 2026-11-26 and 2026-11-27, requests/control 7 of 7 nights, requests/manager 4 of 7)
- https://github.com/tacplus-dev/roadmap/blob/main/suggestions/recap/2026-11-26.md (requests: scan memory 512Mi asked, used p50 1482, p90 2715, max 2766Mi over 38 pods; control CPU 50m asked, used p90 177m; 'No advice tonight' on both)
- https://github.com/tacplus-dev/looper/issues/165 (closed: the pods' requests set per kind in looper-dispatch's LOOPER_REQUESTS_<kind>, and the recap's requests-against-use section that measures them)
- https://github.com/tacplus-dev/roadmap/blob/main/decisions/looper-build-vms-2026-11-24.md (Migration plan, steps 4 to 6: the in-cluster control runner stays until the managers move and the cluster shrinks; 'Superseded: no new work on in-cluster runner sizing' names cluster#53's node sizing, not the pods' requests)

## Cost
Minutes: three values in ci/tacplus-dev.gitlab-ci.yml (looper-dispatch's LOOPER_REQUESTS_scan, _control and _manager), landed through the looper's manager and live at the next pipeline the looper project reads; the control runner's concurrent, counted from the pool's room by requests (cluster#36, #64), follows by itself. Nothing to buy: the normal pool was full for 0 of 983 pods tonight, so the right requests cost no node

## Touches
decisions/cluster-node-pools-2026-11-03.md (the normal pool's room), decisions/looper-build-vms-2026-11-24.md (the control runner stays through the migration), looper#165 (requests from measurement), cluster#36 and cluster#64 (control's concurrent from the pool's room), looper#57 and looper#72 ('recap should let us know if we need to adjust')

## Reuse considered
The recap's requests section already measures each kind against its request every night and names the setting to change (looper-dispatch's LOOPER_REQUESTS_<kind>, the CI file's .control): nothing new, a setting read from it. cluster's control.yaml concurrent is counted from the pool's room by the pods' requests (cluster#36), so a true request also makes the slot count true; the open suggestion of 2026-11-16 (a runner of the kick's own) and cluster#60 (the managers' runner) change where the pods run, not what they ask for

## Advice
Recap: 2026-11-27 requests/scan
Cause: looper-dispatch's LOOPER_REQUESTS_scan asks 512Mi of memory for a nightly scan's pod while the scans use p50 1816Mi, p90 2333Mi, max 3204Mi (24 pods tonight; p90 2715Mi the night before), and LOOPER_REQUESTS_control asks 50m CPU against a p90 of 159m (seven nights) while LOOPER_REQUESTS_manager asks 250m against a p90 of 38m: the scheduler packs the normal pool by requests it cannot trust, over-committing memory on the scans and holding CPU idle for the managers
Change: ci/tacplus-dev.gitlab-ci.yml (the looper project's CI file, looper-dispatch's requests): LOOPER_REQUESTS_scan memory 512Mi to 2560Mi with a 3.5Gi limit, LOOPER_REQUESTS_control CPU 50m to 200m, LOOPER_REQUESTS_manager CPU 250m to 50m; the control runner's concurrent recounted from the pool's room (cluster#36, #64) follows
Gain: No nightly scan evicted for memory it never asked for (a scan evicted is a repository with no filing and no proposals that night), the normal pool's room counted from true requests (about 2.4 GiB a scan, not 0.5), and about 200m of CPU a manager pod given back to the pool's count; the item requests/scan and requests/control leave the recap's recurring list

## Status
open
Rank: 5
Filed: 2026-11-27 by the roadmap scan (https://gitlab.com/tacplus-dev/looper/-/jobs/18423975993)
