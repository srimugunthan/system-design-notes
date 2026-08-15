# ML System Design

A comprehensive breakdown of the most commonly asked ML system design questions, organized by category, with the ones mapping most directly to fraud, financial crime, and agentic AI work flagged.

## 🔁 Recommendation Systems

These are the highest-frequency category in FAANG/Big Tech interviews.

- Design a YouTube/Netflix video recommendation system
- Design a "People You May Know" system (LinkedIn/Facebook)
- Design a feed ranking system (Twitter, Instagram)
- Design a music recommendation engine (Spotify)
- Design a product recommendation system (Amazon)

**Core skills tested:** collaborative filtering, two-tower models, retrieval vs. ranking stages, feature stores, real-time vs. batch serving, cold start.

## 🔍 Search & Ranking

- Design a search ranking system (Google/Bing)
- Design a job/candidate matching system (LinkedIn)
- Design a semantic document search system
- Design an ads ranking and auction system

**Core skills tested:** Learning-to-Rank (LambdaMART, ListNet), query understanding, embedding-based retrieval, business objective vs. ML objective alignment.

## 🛡️ Fraud & Anomaly Detection ← Strongest zone

- Design a real-time fraud detection system (credit card transactions)
- Design an AML (Anti-Money Laundering) transaction monitoring system
- Design an account takeover detection system
- Design a bot detection system
- Design an insider threat detection system

**Core skills tested:** concept drift, class imbalance, graph-based models (GNNs), streaming feature engineering, rule+ML hybrid, precision-recall tradeoffs, regulatory constraints.

## 🗣️ NLP / Language Systems

- Design a spam/phishing email classifier
- Design a toxic content detection system (hate speech, NSFW)
- Design a document summarization system
- Design a conversational AI / chatbot
- Design a named entity recognition pipeline
- Design an LLM-powered RAG system

**Core skills tested:** fine-tuning vs. prompting, embedding pipelines, vector databases, hallucination mitigation, latency vs. quality.

## 👁️ Computer Vision

- Design an image moderation system (NSFW, violence)
- Design a facial recognition system
- Design an OCR pipeline for document extraction ← FinVision aligns here
- Design an object detection system for autonomous vehicles
- Design a visual search system (Pinterest Lens)

## 📈 Forecasting & Prediction

- Design a demand forecasting system (inventory/supply chain)
- Design a stock price movement predictor
- Design a customer churn prediction system
- Design a loan default / credit risk model ← AuditAgent aligns here
- Design an ETA prediction system (Uber/Lyft)

**Core skills tested:** time-series handling, feature leakage prevention, calibration, explainability for compliance.

## 🤖 GenAI / LLM Systems (emerging, fast-growing)

- Design an LLM red-teaming and evaluation framework ← RedTeamLoop is this
- Design a RAG pipeline for enterprise document Q&A
- Design a prompt injection defense system ← Shield-Fin is this
- Design an AI agent with tool use and memory
- Design an LLM fine-tuning and evaluation pipeline

**Core skills tested:** LangGraph/orchestration, guardrails, evaluation harnesses, latency-cost tradeoffs, model versioning.

## 📊 Ads & Monetization

- Design a Click-Through Rate (CTR) prediction model
- Design a conversion rate optimization system
- Design a budget pacing system for advertisers
- Design a real-time bidding (RTB) system

## 🏗️ ML Platform / Infrastructure

- Design a feature store
- Design an ML model monitoring system (data drift, concept drift)
- Design an A/B testing platform for ML models
- Design a model registry and deployment pipeline
- Design a data labeling platform

**Core skills tested:** MLflow, shadow deployment, canary releases, champion-challenger, drift detection (PSI, KS test).

## Framework for Answering Any ML System Design Question

Most interviewers expect a walkthrough of these phases:

1. **Problem Framing** — Business objective → ML objective → metric alignment
2. **Data** — Sources, volume, labels, class imbalance, freshness
3. **Feature Engineering** — Raw → derived features, real-time vs. batch, feature store
4. **Modeling** — Baseline → iterations, model family, training strategy
5. **Evaluation** — Offline metrics, online A/B test, business KPIs
6. **Serving & Infrastructure** — Latency, throughput, batch vs. real-time, scaling
7. **Monitoring & Iteration** — Drift detection, retraining triggers, feedback loops

er should go beyond the standard framework and inject specific architectural choices as concrete examples — that's what separates a Principal-level answer from a Senior-level one.
