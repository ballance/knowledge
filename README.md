# Knowledge Repository

Welcome to my personal knowledge repository! This is a curated collection of essential articles, papers, and insights that I believe every software engineer and technology professional should be familiar with. I'm sharing these summaries and resources to help others quickly grasp fundamental concepts that have shaped modern computing.

## Articles & Summaries

### [Real-Time Log Systems](./logs.md)
**Jay Kreps, LinkedIn Engineering**  
A comprehensive summary of "The Log" - exploring log-structured systems as the unifying abstraction for distributed data systems. Covers real-time data pipelines, event sourcing, CQRS patterns, and the observability stack that powers modern distributed services. Essential reading for anyone building or maintaining distributed systems.

### [No Silver Bullet](./no-silver-bullet.md)
**Frederick P. Brooks Jr., 1986**  
Brooks' seminal paper that remains remarkably relevant 40 years later. Argues that no single technology or practice will provide order-of-magnitude improvements in software productivity. The distinction between essential and accidental complexity continues to shape how we think about software engineering challenges and why some problems remain fundamentally hard.

### [Unicode & Character Encoding](./unicode-character-encoding.md)
**Joel Spolsky, 2003**  
The essential guide to understanding character encodings and Unicode. Explains why "plain text" doesn't exist, the evolution from ASCII to Unicode, and practical implementation details every developer needs to avoid encoding bugs. If you've ever seen mojibake (文字化け) in production, this article explains why.

### [Dynamo: Amazon's Highly Available Key-value Store](./dynamo-amazon-highly-available-kvstore.md)
**DeCandia et al., Amazon, 2007**  
The paper that launched the NoSQL movement. Introduces techniques like consistent hashing, vector clocks, and sloppy quorum that became foundational for distributed databases. Shows how Amazon built an "always-on" shopping cart experience by choosing availability over consistency, demonstrating practical CAP theorem trade-offs at massive scale.

### [The Twelve-Factor App](./twelve-factor-app.md)
**Adam Wiggins and Heroku Team, 2011**  
The definitive methodology for building cloud-native applications. These twelve principles address configuration management, dependency isolation, horizontal scaling, and the dev/prod parity gap. Born from running thousands of apps on Heroku, it's become the standard for modern SaaS applications and influenced everything from Docker to Kubernetes.

## Source Materials

Original documents are available for reference:

**PDFs** (in [`/sources`](./sources) directory)
- [What Every Computer Scientist Should Know About Floating-Point Arithmetic](./sources/what-every-computer-scientist-should-know-about-floating-point-arithmetic.pdf) - David Goldberg's comprehensive guide to floating-point representation and arithmetic
- [No Silver Bullet](./sources/no-silver-bullet-brooks-1986.pdf) - Brooks' original 1986 paper

**HTML Archives**
- [The Log - LinkedIn Engineering](./the-log-linkedin-engineering.html) - Full article by Jay Kreps

## Contributing

This repository represents my personal learning journey and the knowledge I find most valuable. While it's primarily for my own reference and sharing, I welcome discussions and suggestions for other fundamental texts that have shaped our field.

## License

See [LICENSE](./LICENSE) for details.