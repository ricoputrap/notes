---
name: sdd-phase3.5-quantitative
description: Calculate performance metrics, capacity planning, and infrastructure costs for quantitative analysis
argument-hint: [design-file-path]
user-invocable: true
---

# Phase 3.5: Quantitative Analysis Generation Skill

Generate quantitative analysis covering performance metrics, capacity planning, and infrastructure costs.

## How to Use This Skill

**Invoke manually:**
```
/sdd-phase3.5-quantitative
/sdd-phase3.5-quantitative path/to/design-doc.md
```

**What AI will do:**
1. Read your Design Doc and Requirements
2. Ask about expected scale (users, daily actives, growth rate)
3. Calculate QPS (requests per second) and latency budgets
4. Estimate storage, memory, and CPU requirements
5. Calculate infrastructure costs (cloud provider pricing)
6. Define scaling decision points and triggers
7. Provide output ready for Phase 4 (Swagger Doc)

## What Constitutes Good Quantitative Analysis

Your analysis should:
- **Be based on realistic assumptions:** Know your user base size and growth
- **Have measurable targets:** Numbers, not vague words
- **Identify bottlenecks:** Where will the system hit limits?
- **Define scaling points:** When and how to scale
- **Align with budget:** Infrastructure costs match business reality
- **Be documented:** Assumptions clear for future reference

## Output Format

The generated analysis will include:

### 1. Performance Metrics
```
Expected Scale:
- Users: 10,000
- Daily active: 50% = 5,000/day
- Actions per user/day: 10
- Daily requests: 50,000

QPS Calculation:
- Average: 50,000 / 86,400 seconds ≈ 0.58 QPS
- Peak (10x during business hours): 5.8 QPS
- Database QPS (2 queries/request): 11.6 QPS

Latency Budget (95th percentile target: 200ms):
- Network: 20ms
- API processing: 50ms
- Database query: 100ms
- Serialization: 30ms
- Total: 200ms ✓

Bottleneck Analysis:
- Database is bottleneck (100ms of 200ms)
- Query optimization critical
- No caching needed at this scale
```

### 2. Capacity Planning
```
Storage:
- Users: 10,000 × 1KB = 10MB
- Tasks: 100,000 × 1.5KB = 150MB
- Comments: 500,000 × 0.5KB = 250MB
- Total: ~410MB

Growth:
- Monthly: 125MB (tasks + comments)
- Yearly: 1.5GB
- 3-year: 4.91GB (fits on single database)

Server Resources:
- CPU: 2 cores (5.8 QPS requires <5% CPU)
- Memory: 2GB (Node.js + connections + buffer)
- Storage: 5GB database volume

Concurrent Users:
- Peak: 1,000 concurrent
- Connection pool: 20 (sufficient with pooling)
```

### 3. Cost Analysis
```
Cloud Infrastructure (AWS Example):

Compute: $15.18/month
Database: $13.56/month
Monitoring: $65/month
Total: $93.74/month = $1,124.88/year
Cost per user: $0.009/month

At 10x Scale:
- Compute: $30.36/month (2 instances)
- Database: $30.41/month (larger instance)
- Load balancer: $16.20/month
- Networking: $50/month
- Total: $376.97/month = $4,523.64/year
```

### 4. Scaling Decision Points
```
If QPS > 10:
- Add load balancer ($16.20/month)
- Add 2nd application server ($15.18/month)
- Cost increase: $31.38/month

If Storage > 100GB:
- Archive old data to S3
- Implement table partitioning
- Cost reduction: ~$23/GB saved

If Concurrent > 1,000:
- Add database read replicas
- Implement session caching (Redis)

If Cost > Budget:
- Use reserved instances (30% savings)
- Archive historical data
- Optimize queries
```

## Instructions for Claude

When the user invokes this skill:

1. **Obtain the Design Doc**
   - If file path provided: Read it
   - If not provided: Ask user to paste or provide path
   - Also read Requirements for constraints

2. **Clarify Expected Scale**
   - Ask: "How many users do you expect?"
   - Ask: "What's your daily active user percentage? (e.g., 50%)"
   - Ask: "Average actions per user per day?"
   - Ask: "Expected growth rate? (e.g., 50%/year)"
   - Ask: "Timeline and scale projections? (1 year, 3 years?)"

3. **Calculate Performance Metrics**
   - Convert daily requests to QPS (requests per second)
   - Separate average QPS from peak QPS
   - Estimate database QPS (multiply by avg queries per request)
   - Allocate latency budget across layers:
     - Network latency
     - API processing time
     - Database query time
     - Response serialization
   - Identify bottlenecks (which component takes most time?)
   - Check if design can handle peak load (should have margin)

4. **Estimate Capacity Needs**
   - **Storage:** Calculate per entity (users, tasks, comments, etc.)
     - Bytes per record × number of records
     - Project growth over 1, 3, 5 years
     - Note when current design hits limits
   - **Memory:** Node.js baseline + connections + cache
     - Connection pool needs (typical 20-50)
     - Cache if needed (estimate data size)
   - **CPU:** Estimate cores needed based on QPS
     - Modern CPUs handle 100+ QPS per core
   - **Concurrency:** Calculate concurrent users from daily actives
     - Daily actives × peak concurrency percentage

5. **Calculate Infrastructure Costs** (assume AWS)
   - **Compute:** EC2 instance hourly rate × 730 hours/month
     - Small workloads: t3.small ($0.0208/hr)
     - Medium: t3.medium ($0.0416/hr)
     - Large: m5.large ($0.096/hr)
   - **Database:** RDS PostgreSQL instance + storage
     - Micro: $0.017/hr
     - Small: $0.034/hr
     - Storage: $0.23/GB/month
   - **Monitoring:** CloudWatch logs, APM tools
     - CloudWatch: $0.50/GB
     - DataDog/NewRelic: ~$50-100/month
   - **Networking:** Data transfer, load balancer
     - LB: $16.20/month + data transfer
   - **Total:** Monthly and annual costs
   - **Cost per user:** Total cost ÷ number of users

6. **Project Multi-Year Costs**
   - Year 1: Initial setup costs
   - Year 2: Scaling scenario 1 (e.g., 5x growth)
   - Year 3: Scaling scenario 2 (e.g., 10x growth)
   - Show cost progression

7. **Define Scaling Decision Points**
   - At what metric does each component need to scale?
   - What architectural changes are needed?
   - What's the cost impact of each scaling step?
   - Examples:
     - "If QPS > 50: Add load balancer + 2nd server (+$31.38/mo)"
     - "If storage > 100GB: Archive to S3 (saves $23/GB)"

8. **Validate the Analysis**
   - Assumptions are documented
   - Numbers are realistic for the domain
   - Design can handle projected growth
   - Costs align with business reality
   - Scaling points are clear and actionable

9. **Provide Output** as markdown document:
   - Organized sections (metrics, capacity, costs, scaling)
   - All assumptions documented
   - Ready to commit
   - Informs Swagger design (latency budgets matter for endpoint design)

10. **Ask for Feedback**: "Do these numbers align with your expectations? Any assumptions to adjust? Ready for Phase 4 (Swagger Doc)?"

## Example Interaction

```
User: /sdd-phase3.5-quantitative

Claude: I'll calculate performance metrics, capacity, and costs from your Design Doc.

First, let me understand your expected scale:
1. How many users do you anticipate? (e.g., 10,000)
2. What % are daily active users? (e.g., 50%)
3. Average actions per user per day? (e.g., 10)
4. Expected yearly growth rate? (e.g., 50%)

[User provides numbers]

Claude: Perfect! Here's your Quantitative Analysis:

# Task Management API - Quantitative Analysis

## Performance Metrics
- QPS: 0.58 average, 5.8 peak
- Latency budget: 200ms (database is bottleneck at 100ms)
- Bottleneck: Database queries (optimize with indexes)

## Capacity Planning
- Initial storage: 410MB
- 3-year storage: 4.91GB
- Server: 2GB RAM, 2 CPU cores
- Concurrent users: 1,000 at peak

## Cost Analysis
- Initial: $93.74/month = $1,124.88/year
- At 10x scale: $376.97/month = $4,523.64/year
- Cost per user: $0.009/month

## Scaling Points
- QPS > 10: Add load balancer + 2nd server (+$31.38/mo)
- Storage > 100GB: Archive old data
- Concurrent > 1,000: Add database read replicas

This analysis validates your design can handle expected scale.
Ready for Phase 4: Swagger Doc generation!
```

## Tips for Best Results

- **Get scale assumptions right:** Base on realistic user numbers
- **Include growth projections:** Don't assume scale stays constant
- **Use actual cloud pricing:** AWS, Azure, GCP have calculators
- **Identify bottlenecks early:** Optimize expensive operations first
- **Document assumptions:** "Assuming 50% daily active users" helps later
- **Plan for growth:** What happens at 5x, 10x, 100x scale?

## Related Skills

After completing analysis:
- Next: `/sdd-phase4-swagger` - Generate Swagger/OpenAPI spec
- Previous: `/sdd-phase3-design` - Generate Design Doc
- Reference: [Spec-Driven Development Workflow](../../spec-driven-development-workflow.md) - Phase 3.5 details
