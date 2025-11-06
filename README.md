# AWS DNS Outage (20 October 2025) Technical Analysis & Lessons Learned

# Date
October 20 2025  
Region: US-EAST-1 (Northern Virginia)

--------------------------------------------
# Summary

On October 20, 2025, several AWS services started showing higher error rates and increased latency after a DNS resolution failure occurred inside AWS’s internal systems that support Amazon DynamoDB in the US-EAST-1 (N. Virginia) region. 
The issue began at around 9:00 AM (SAST) on October 20 and was fully resolved by midnight the same day.

Although the underlying defect was relatively small, it caused a widespread chain reaction that affected multiple AWS services and hundreds of applications around the world. This incident was a clear reminder of how even a minor automation glitch in a large distributed cloud platform can have global consequences.

# Timeline (SAST)
  Time-Event:

8:48 AM (Oct 20) Increased error rates and latency across several AWS services in US-EAST-1.                                        
9:26 AM (Oct 20) Root cause identified as DNS resolution failures for DynamoDB regional endpoints.                                  
11:24 AM (Oct 20) AWS mitigated the DNS issue and began restoring service.                                                          
3:01 AM (Oct 21) AWS declared all services fully operational.                                                                       


Sources: [AWS Status Page](https://status.aws.amazon.com/), The Register, Wired, TechCrunch Engadget.                              
# https://www.wired.com/story/aws-cloud-outage-long-tail/                                                                           
# https://www.theregister.com/2025/10/23/amazon_outage_postmortem/                                                                  

# Root Cause (Simplified)

   AWS operates an automated system to manage DNS records for regional service endpoints.  
   Two internal components Planner and Enactor coordinate updates to DNS entries.

   A race condition occurred between two Enactor processes:
   Enactor A was processing an older plan.
   Enactor B completed a newer plan and cleaned up the stale one.
   During cleanup, all IP address records for the regional DynamoDB endpoint were accidentally deleted.
   Result → DNS returned NO IPs for DynamoDB → clients couldn’t connect → dependent services failed.
 
# Cascading Impact

   Directly affected:
   The issue originated with the DynamoDB API in the US-EAST-1 region, where DNS resolution failures prevented normal operations.

# Key Technical Lessons:
   DNS can become a single point of failure => Even a small DNS automation bug can break communication with critical AWS services.

   Too much dependency on one region increases risk => Many AWS systems still rely heavily on US-EAST-1, so issues there can quickly spread across other regions.

   Automation needs safety checks => Any automated cleanup or update process should include proper validation and rollback mechanisms.


----------------------------

  # Small Diagram — Simplified Failure Chain:

      DNS Automation Bug                                                                                                                
           ↓
      DynamoDB Endpoint Unresolvable                                                                                                    
            ↓
      Service Requests Fail                                                                                                             
            ↓
      Dependent AWS Services Impacted                                                                                                   
            ↓
      Global App Outages                                                                                                                

------------------------------------------
  # Root Cause of AWS Issue:-

   Why Multi-AZ or Multi-Region Didn’t Help in This AWS Outage:

   Well !! Some of the junior engineers or people not directly involved with cloud technologies asked me: -
   What is the point of using Multi-AZ or Multi-Region setups if our applications still went down why ?

   Let me clarify this point clearly:-

   Multi-AZ or Multi-Region configurations didn’t help during this particular AWS outage because High Availability (HA) only protects against infrastructure-level Or regional failures not global DNS resolution issues.
   When we design for high availability in AWS => Multi-AZ (Availability Zone) redundancy protects against zone-level failures (exmp:- power loss Or hardware issues in one data centre).

   Multi-Region setups protect against regional outages (exmp:  if us-east-1 goes down, you can fail over to eu-west-1 or us-west-2).
   However,during the October 2025 outage, the failure occurred in AWS internal DNS infrastructure, which operates above all regions and zones.

   This means Even if workloads were distributed across multiple AZs or regions, they still couldn’t reach AWS service endpoints such as DynamoDB or internal APIs, because DNS itself wasn’t responding correctly.
   I hope this explanation makes the concept clear. it wasn’t our application architecture that failed, but the global name resolution layer that everything depends on.

# Lesson Learned from this:
   High Availability protects against infra or regional outages, not global control-plane failures.  
   Implement DNS-independent failover paths where possible.  
   Multi cloud redundancy (Exp: AWS & Azure) can reduce exposure to single provider DNS issues.
