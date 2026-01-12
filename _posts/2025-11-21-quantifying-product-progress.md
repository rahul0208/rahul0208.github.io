---
layout: post
title: "From Incomplete to Market-Ready: Quantifying Product Progress"
tags: [Management, Product]
---
Over the last two years, I was in pursuit of a new product. We used an iterative approach, delivering and improving features in every cycle. Because the product was aspirational in nature, every feature often required updates or changes after the first cut. I was leading a technology team tasked with delivering these business aspirations, but a significant challenge emerged: while the tech team operated in short iterations, the business stakeholders operated on yearly delivery targets. This discrepancy created a **chasm** in our operations several times during our journey.

### The Challenge: The "Incomplete" Trap

The challenge stemmed from the uncertainty of feature requirements. The business intended to have features fully in place by the end of their yearly targets, but as a tech team, we were in constant flux trying to determine when a feature could be marked as "complete."

Because of this, we were unable to prove that significant progress had been made during the first six quarters. Despite delivering numerous workflows, everything was tagged as "Incomplete" by the business. This led to low team morale, as developers were unsure of the delivery targets for each quarter. Simultaneously, business stakeholders questioned the delivery quality because they couldn't see objective progress against their yearly goals.

We realized the problem wasn't a lack of development—it was a **lack of measurement.** The iterative model allowed us to showcase features early for feedback, but without a clear definition of "done," this feedback simply led to endless rework.

### The Solution: Quantifying Maturity through QA

To bridge this gap, we transformed our QA process into a data engine for the business. We took two critical steps:

1. **Independent Documentation**
   QA worked directly with stakeholders to document every conceivable scenario as an independent test case—creating an objective view of what the business actually expected.

2. **Accepting Failure Early**
   We normalized the idea that test cases could fail or remain unsupported in early iterations. Failure wasn’t a defect; it was information.

Next, we defined **feature completion as a percentage of successfully executed test cases.** This provided an objective number to quantify if a feature was "good enough" or needed more work. We found our features typically varied between 67% and 93% completion.

This number became our **Guiding Star**. We discovered that a complex feature with many workflows might be viable for delivery at 60% completion, whereas a smaller, simpler workflow might require 75%. These percentages helped us clear the uncertainty surrounding our targets. Most importantly, it allowed us to gauge the overall **maturity of the product**, helping us determine a realistic "Go-To-Market" (GTM) application maturity target.

### The Result: A Data-Driven Roadmap

Failed or unsupported test cases stopped being "bugs" and started being "data." They highlighted "unknowns" that required more thought. This provided the Product Owner with the fodder needed to classify whether a specific use case was actually required or if it could be de-scoped to meet deadlines.

By using test data to measure maturity rather than just tracking defects, we gave the business the confidence to sell the product. In the following four quarters, we delivered a stable product. It didn't support 100% of every imagined scenario, but it successfully covered **77% of core business scenarios.** This was enough to bring in customers, learn from real usage, and justify investing in the remaining edge cases—**with data, not assumptions**.