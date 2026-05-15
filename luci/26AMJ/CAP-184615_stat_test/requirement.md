Currently we are working to persist the CouponSeriesStatisticsField from Redis to MySql DB via todays summary in stats_series_summary and till yesterday i.e ETLed data from DataBricks in stats_history_blueprint table, stats_history_registry tables. 

We are doing this to avoid making aggregate queries to compute these stats kets during pod startup, when coupon series config updates, on coupon upload notication...etc . via com.capillary.luci.api.config.CouponSeriesStatisticsService#calculateAllStats

At present, we released following code changes
1. Writing to both redis and mysql i.e increment the stats counters.
2. Data bricks notebook has been setup to create store series level stats from ETLd data bricks  table in S3. Daily RMQ message triggers com.capillary.luci.jobs.runner.StatsHistoryRunner#run to download this from S3 to mysql db.
3. Reading the limits from mysql instead of redis during limit checks in write APIs and in GET apis.
4. We have setup an cron to valdiate these data on daily basis
        com.capillary.luci.jobs.runner.StatsCounterDiffCheckerJob#runDiffCheck
        com.capillary.luci.jobs.runner.StatsCounterDiffCheckerJob#runDiffCheck
        
        

Concern:Need to ensure there are proper test UT, IT and Regression coverage w.r.t stats interms of limit check and stats exposed in GET Apis responsed. Most cases looks to be testing by creating a new series, which wont cover stats from history.

Whats needed:
IT test coverage
- Go over existing ITs, UTs compare with implementation and identify test coverage gaps.
- Validate if any gaps in the implementation or corner/boundy case handling.
- Most cases Look for ways to test stats values from Hitstory + summary table i.e for long series which is typical.

Automated test coverge:
/Users/saravanankesavan/sara/wsgit/campaigns_auto
Above repo is an python based API tester gets executed as test suites sanity, smoke and resgression via schedule by running them in a dedicated pod in product clusters within same VPC. More of a black box testing.
All tests related to Luci are kept in /Users/saravanankesavan/sara/wsgit/campaigns_auto/tests/luci.

- Analyze if proper assertions w.r.t stats are in place to validate MySql based stats.
- Solution how to setup an series that runs for a longer duration across days and assert the stats so it tests history data. But all current tests creates a new series.
