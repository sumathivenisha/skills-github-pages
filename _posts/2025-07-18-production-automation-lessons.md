---
layout: post
title: "Automation Lessons from the Trenches: What Breaks in Production"
date: 2025-07-18 14:00:00 +0400
categories: [Automation, Production, Lessons-Learned]
tags: [rpa, automation, production-issues, n8n, power-automate]
author: Sumathi Malikarjun
excerpt: Seven years, dozens of failures, and countless production incidents. Here are the hard lessons that separate working automations from scalable systems.
---

## The Enthusiasm-to-Reality Gap

Everyone gets excited about automation ROI:

**Sales pitch:** "Automate this 2-hour task → save 8 hours/week → $50K/year savings"

**Production reality:** The task is 2 hours 90% of the time. The other 10%? Edge cases that break everything.

You're now paying someone $30/hour to babysit an automation that was supposed to be hands-off.

<!--more-->

## Lesson 1: Edge Cases Will Destroy You

### The Problem

Your automation works perfectly for the happy path:

```
Input: Well-formatted invoice
Process: Extract, validate, post
Result: ✓ Success
```

Then production hits:

```
Input: Invoice with handwritten notes
       Invoice with attachment
       Invoice from new vendor (extra fields)
       Invoice dated 2020 but uploaded today
       Invoice for $0.01
       Invoice missing PO number
Result: ✗✗✗ Automation breaks silently
```

### My Lesson

After 6 months of silent failures consuming exception handling time, I built **explicit edge case detection**:

```python
class InvoiceValidator:
    def validate_before_processing(self, invoice):
        """
        Find edge cases BEFORE they break the automation.
        """
        edge_cases = []
        
        # Check 1: Date sanity
        if invoice.date > datetime.now():
            edge_cases.append({
                "type": "future_date",
                "value": invoice.date,
                "action": "escalate_to_human"
            })
        
        # Check 2: Amount sanity (too small, too large, zero)
        if invoice.amount < 0.01:
            edge_cases.append({
                "type": "suspiciously_small_amount",
                "action": "require_approval"
            })
        
        if invoice.amount > self.AMOUNT_THRESHOLD:
            edge_cases.append({
                "type": "large_amount",
                "action": "require_approval"
            })
        
        # Check 3: Vendor validation
        if not self.is_known_vendor(invoice.vendor):
            edge_cases.append({
                "type": "unknown_vendor",
                "action": "escalate_and_learn"
            })
        
        # Check 4: Document quality
        if self.has_handwritten_text(invoice):
            edge_cases.append({
                "type": "handwritten_content",
                "action": "use_ocr_review_human"
            })
        
        if not self.has_po_reference(invoice) and \
           invoice.amount > self.AMOUNT_THRESHOLD:
            edge_cases.append({
                "type": "missing_po_reference_large_amount",
                "action": "escalate"
            })
        
        return {
            "is_safe": len(edge_cases) == 0,
            "edge_cases": edge_cases,
            "recommended_action": self.decide_action(edge_cases)
        }
```

**Result:** 0 silent failures. All edge cases go to human review queue. System stays trustworthy.

---

## Lesson 2: Time Dependency Kills Automations

### The Problem

Your automation works every day at 9 AM. Then:

- Weekend runs fail (external system down)
- Month-end runs timeout (10x volume)
- Holiday runs crash (batch process skipped, causing backlog)
- Daylight saving time shifts everything

Your automation isn't robust. It's just lucky on normal days.

### My Lesson

For **time-dependent automations**, build adaptive logic:

```python
class RobustScheduledAutomation:
    def should_run_today(self):
        """
        Don't always run at the scheduled time.
        Check if conditions are right.
        """
        checks = {
            "external_systems_healthy": self.health_check(),
            "not_holiday": not self.is_holiday(),
            "not_peak_load": not self.is_peak_processing_time(),
            "previous_run_completed": self.check_previous_run_status(),
            "queue_not_backed_up": self.check_queue_health()
        }
        
        if all(checks.values()):
            return {"should_run": True, "reason": "all_checks_pass"}
        else:
            failed = [k for k, v in checks.items() if not v]
            return {
                "should_run": False,
                "reason": f"Failed: {failed}",
                "action": "defer_and_notify"
            }
    
    async def run_with_backpressure(self):
        """
        Don't run all at once. Spread load over time.
        """
        items_to_process = await self.get_queue()
        
        if len(items_to_process) > self.NORMAL_BATCH_SIZE:
            # Backpressure: lots of items waiting
            # Process in smaller batches to not overwhelm downstream
            batch_size = self.NORMAL_BATCH_SIZE // 2
            delay_between_batches = 60  # seconds
        else:
            batch_size = self.NORMAL_BATCH_SIZE
            delay_between_batches = 0
        
        for i in range(0, len(items_to_process), batch_size):
            batch = items_to_process[i:i+batch_size]
            await self.process_batch(batch)
            
            if delay_between_batches > 0:
                await asyncio.sleep(delay_between_batches)
```

**Result:** Automations self-heal based on system state, not just scheduled time.

---

## Lesson 3: Silent Failures Are Your Biggest Enemy

### The Problem

Automation runs. Returns "success". No human ever checks.

2 months later, you discover none of the data was actually posting to the backend system.

2 months of bad data. 2 months of decisions made on incomplete information.

### My Lesson

**Every automation must validate its own work:**

```python
class AutomationWithSelfValidation:
    async def process_and_validate(self, input_data):
        """
        Process, then verify what actually happened.
        """
        
        # Step 1: Process
        result = await self.process(input_data)
        
        # Step 2: CRITICAL - Validate work was done
        validation = await self.validate_work_completed(
            input_data,
            result
        )
        
        if not validation.success:
            # Process said success, but validation failed
            # This is a CRITICAL ALERT
            await self.alert_critical({
                "issue": "Process-validation mismatch",
                "what_happened": result,
                "what_should_have_happened": validation.expected,
                "difference": validation.error_details,
                "action": "escalate_immediately"
            })
            
            # Don't just return success
            return {
                "process_result": result,
                "validation_result": validation,
                "status": "FAILED_VALIDATION - see alerts"
            }
        
        return {"status": "success", "data": result}

    async def validate_work_completed(self, input_data, process_result):
        """
        Query the downstream system to confirm data was actually posted.
        """
        # If you posted an invoice, query the accounting system
        # If you created a user, query the directory service
        # If you sent an email, check the sent folder
        
        posted_data = await self.query_downstream_system(
            process_result.id
        )
        
        if not posted_data:
            return {
                "success": False,
                "error_details": f"Posted data not found in system (ID: {process_result.id})"
            }
        
        # Compare: did the data arrive intact?
        differences = self.compare_data(input_data, posted_data)
        if differences:
            return {
                "success": False,
                "error_details": f"Data mismatch: {differences}"
            }
        
        return {"success": True}
```

**Rule:** If you can't verify it happened, don't trust it happened.

---

## Lesson 4: Scaling Breaks Different Things

### The Problem

Your automation handles 100 items/day perfectly. Then business grows.

Now it's 5,000 items/day.

Everything breaks:
- API rate limits → You're calling 50x more often
- Database queries → 50x slower now
- Email notifications → Spam folder (you sent 5,000 emails)
- Timeout issues → Task that took 30 seconds now takes 30 minutes

### My Lesson

**Design for 10x load from day one:**

```python
class ScalableAutomation:
    def __init__(self):
        # Concurrent processing, not sequential
        self.max_concurrent = 10
        self.batch_size = 100
        
        # Circuit breaker: if downstream system is struggling, back off
        self.rate_limiter = ExponentialBackoffRateLimiter(
            initial_rate=100,  # calls per minute
            max_wait=300,  # seconds
        )
        
        # Queue for retries instead of failing immediately
        self.retry_queue = PersistentQueue()
    
    async def process_at_scale(self, items):
        """
        Process many items efficiently.
        """
        # Don't process sequentially
        # Don't send 5,000 individual API calls
        
        # Instead: Batch + Concurrent
        for batch in self.chunk(items, self.batch_size):
            # Process up to 10 items concurrently
            tasks = [
                self.rate_limiter.call(
                    self.process_item,
                    item
                )
                for item in batch
            ]
            
            results = await asyncio.gather(*tasks, return_exceptions=True)
            
            # Handle failures gracefully
            for item, result in zip(batch, results):
                if isinstance(result, Exception):
                    # Add to retry queue, don't fail entire batch
                    await self.retry_queue.add(item)
                else:
                    await self.log_success(item, result)
    
    async def handle_rate_limiting(self):
        """
        External system: "slow down, you're calling too fast"
        Response: automatically back off
        """
        self.rate_limiter.back_off(
            multiplier=2.0,  # Slow down 2x
            max_wait=300
        )
        await self.notify_ops(
            "Rate limited by downstream system. Backing off."
        )
```

**Result:** Automations that handle 100 items handle 5,000 items. Different class problems.

---

## Lesson 5: Alerting Strategy Matters

### The Problem

You get 100 alerts per day. Most are false positives.

You stop reading alerts. One day, a critical one is buried.

### My Lesson

**Build a 3-tier alerting strategy:**

```python
class AlertingStrategy:
    def alert(self, issue):
        """
        Route alerts based on severity, not volume.
        """
        
        if issue.severity == "CRITICAL":
            # Critical: wake someone up
            await self.call_oncall_engineer(issue)
            await self.slack_channel(issue, mention="@here")
            await self.page_incident_commander()
        
        elif issue.severity == "HIGH":
            # High: send email, add to dashboard, but don't page
            await self.send_email(issue)
            await self.slack_channel(issue, mention="@team")
            await self.dashboard_widget.update(issue)
        
        elif issue.severity == "MEDIUM":
            # Medium: log for daily review
            await self.log_to_operations_dashboard(issue)
            # Daily digest, not real-time alert
        
        elif issue.severity == "INFO":
            # Info: just log it
            await self.write_to_log(issue)

class CriticalIssueDetector:
    """
    Only truly critical things should be CRITICAL.
    """
    
    def assess_severity(self, issue):
        """
        Is this actually critical?
        """
        if issue.type == "silent_failure":
            # DATA LOSS → CRITICAL
            return "CRITICAL"
        
        elif issue.type == "rate_limit":
            # Slowing down, but working → HIGH
            return "HIGH"
        
        elif issue.type == "single_item_failed":
            # One item failed, retrying → MEDIUM
            return "MEDIUM"
        
        elif issue.type == "informational":
            # Just noting it for auditing → INFO
            return "INFO"
```

**Result:** Engineers actually read critical alerts. False alarm rate drops 90%.

---

## Lesson 6: Error Handling Strategy

### The Problem

You catch all exceptions and log them. Smart, right?

Then a bug propagates silently for 3 months because the exception was logged but never reviewed.

### My Lesson

**Different exceptions need different handling:**

```python
class StratifiedErrorHandling:
    async def process_with_smart_errors(self, item):
        """
        Not all errors are the same.
        Handle them differently.
        """
        
        try:
            # Try to process
            result = await self.risky_operation(item)
            return result
        
        except ValidationError as e:
            # Expected error: data doesn't meet requirements
            # Handle gracefully, escalate if needed
            await self.log_validation_failure(item, e)
            
            if self.should_escalate(item):
                await self.escalate_to_human_review(item)
            
            return {"status": "validation_failed", "reason": str(e)}
        
        except TimeoutError as e:
            # Transient error: retry with backoff
            await self.retry_queue.add(item)
            return {"status": "timeout_retry", "retry_at": self.calculate_retry_time()}
        
        except RateLimitError as e:
            # Rate limit: back off system-wide
            await self.rate_limiter.back_off()
            await self.retry_queue.add(item)
            return {"status": "rate_limited_retry"}
        
        except AuthenticationError as e:
            # Auth failed: might be credential rotation issue
            await self.check_and_refresh_credentials()
            await self.alert_high("Authentication failed, retrying...")
            await self.retry_queue.add(item)
        
        except Exception as e:
            # Unexpected error: this is bad
            await self.alert_critical(
                f"Unexpected error: {type(e).__name__}: {str(e)}"
            )
            await self.escalate_to_human_review(item)
            raise  # Re-raise so monitoring catches it
```

**Rule:** Errors should tell you something. Handle them accordingly.

---

## Lesson 7: Testing Automations is Hard (But Non-Negotiable)

### The Problem

You can't test against the real system (might corrupt real data).
You can't test everything because production has infinite edge cases.

### My Lesson

**Build a test automation that mirrors production:**

```python
class AutomationTestingStrategy:
    async def test_in_staging(self):
        """
        Mirror production as closely as possible.
        """
        staging_data = await self.generate_realistic_test_data()
        
        # Test 1: Happy path
        result = await self.automation.process(staging_data.normal_invoice)
        assert result.status == "success"
        
        # Test 2: Edge cases (ones you know about)
        test_cases = [
            staging_data.invoice_with_handwriting,
            staging_data.invoice_future_dated,
            staging_data.invoice_zero_amount,
            staging_data.invoice_huge_amount,
            staging_data.invoice_missing_po,
            staging_data.invoice_from_new_vendor,
        ]
        
        for test_case in test_cases:
            result = await self.automation.process(test_case)
            # Should not crash, should escalate properly
            assert result.status in ["success", "escalated"]
        
        # Test 3: Load test
        many_items = await self.generate_batch(count=5000)
        start = time.time()
        results = await self.automation.process_batch(many_items)
        elapsed = time.time() - start
        
        assert elapsed < 300, f"Took {elapsed}s for 5000 items"
        assert sum(1 for r in results if r.status == "success") > 0.95 * len(many_items)
```

---

## Lesson 8: Observability is Not Optional

### The Problem

Your automation runs for 6 months. Then it breaks. You have no idea when, why, or how much data was affected.

### My Lesson

**Log everything that matters:**

```python
class FullObservability:
    async def log_automation_execution(self, item, result):
        """
        Log enough to understand what happened.
        """
        await self.logging_db.insert("automation_runs", {
            "item_id": item.id,
            "timestamp": datetime.now(),
            "status": result.status,
            "processing_time_ms": result.elapsed_ms,
            "tokens_used": result.tokens_used,
            "cost_usd": result.cost,
            "error": result.error if result.status == "failed" else None,
            "escalated": result.escalated,
            "human_reviewed": False,  # Filled in later
            "outcome": None,  # Filled in after human review
        })
    
    async def daily_operations_report(self):
        """
        Every morning, understand automation health.
        """
        stats = await self.logging_db.query("""
            SELECT
                DATE(timestamp) as date,
                status,
                COUNT(*) as count,
                AVG(processing_time_ms) as avg_time_ms,
                SUM(cost_usd) as total_cost,
                COUNT(CASE WHEN escalated THEN 1 END) as escalations
            FROM automation_runs
            WHERE DATE(timestamp) = CURDATE()
            GROUP BY 1, 2
        """)
        
        await self.email_ops_team(
            subject="Automation Health Report",
            content=self.format_report(stats)
        )
        
        # Alert if escalations spike
        if stats.escalations > stats.count * 0.05:  # >5% escalation rate
            await self.alert_high("Escalation rate above 5%")
```

---

## Results After Applying These Lessons

| Problem | Before | After | Change |
|---------|--------|-------|--------|
| **Silent Failures** | Regular | ~0/month | Eliminated |
| **Edge Case Crashes** | 8-12/month | 0-1/month | 90% fewer |
| **Manual Fixes** | 20 hrs/week | 2 hrs/week | 90% reduction |
| **System Downtime** | 3 hrs/month | <15 min/month | 95% less |
| **Data Integrity Issues** | 2-3/quarter | 0 | Prevented |
| **Team Confidence** | Low ("automation broke again") | High | Trustworthy |

---

## Key Takeaways

1. **Edge cases are 90% of real automations.** Design for them explicitly.
2. **Time-dependent automations need adaptive logic.** Don't assume conditions are perfect.
3. **Self-validation is non-negotiable.** Never trust the process reported success.
4. **Design for 10x load from day one.** Scaling breaks different things.
5. **Alerting strategy matters.** Route by severity, not volume.
6. **Error types matter.** Handle validation errors differently from timeouts.
7. **Test in staging.** Mirror production as closely as possible.
8. **Observability enables confidence.** Log, measure, report daily.

---

## Final Thought

The best automations are boring. They run silently, reliably, auditably. Nobody celebrates them because they never break.

That's the goal.

---

**Built automations that went sideways?** I'd love to hear your war stories. [LinkedIn](https://linkedin.com/in/sumathi-m-5198271a0).

---

*Seven years, dozens of production automations, countless failures that became lessons.*
