# Bias, Vagueness, and Overload in Prompt Engineering

These are three common reasons prompts produce weak answers.

## 1. Bias

Bias happens when the prompt pushes the model toward a specific answer instead of asking it to reason fairly.

Bad:

```text
Explain why microservices are better than monoliths.
```

This assumes microservices are better.

Better:

```text
Compare microservices and monoliths for a small engineering team.
Explain when each is a better choice.
```

Good prompts leave room for trade-offs.

## 2. Vagueness

Vagueness happens when the prompt does not say what kind of answer is needed.

Bad:

```text
Explain caching.
```

The model does not know the level, goal, or angle.

Better:

```text
Explain caching to a backend developer.
Focus on database load, latency, cache invalidation, and failure modes.
Use one e-commerce example.
```

Specific prompts produce more useful answers.

## 3. Overload

Overload happens when the prompt asks for too many things at once.

Bad:

```text
Teach me Kubernetes, Docker, networking, CI/CD, observability, security, scaling, and system design in one answer.
```

The answer will become shallow.

Better:

```text
Teach me Kubernetes services first.
Explain ClusterIP, NodePort, and LoadBalancer with examples.
End with when each one is used.
```

Break large learning goals into smaller prompts.

## Simple Rule

Before sending a prompt, check:

```text
Bias: Am I forcing the answer?
Vagueness: Did I explain what I need?
Overload: Am I asking too much at once?
```

Better prompts are fair, specific, and focused.
