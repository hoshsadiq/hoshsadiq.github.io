# Conference Talks

Talks I've given or co-presented at conferences and meetups, mostly about infrastructure, platform engineering, and distributed systems.

{{< callout type="info" >}}
Most talks include video recordings and downloadable slides. Source code and demos are available where applicable.
{{< /callout >}}

## 2025

### Building Dynamic Configuration into Terraform

{{<badge content="USENIX SREcon EMEA 2022">}} {{<badge content="Infrastructure as Code" color="blue">}} {{<badge content="Platform Engineering" color="green">}}

**Amsterdam, NL • October 2022**

Exposing opinionated entry points that let developers and managers update infrastructure and services without direct access to Terraform code or repositories is easier than ever with feature flags.

Running Terraform at scale, we reduced friction by separating configuration inputs from implementation. We moved selected Terraform variables into a controlled external interface, letting platform engineers make targeted changes without triggering a full release or apply cycle.

**Key takeaways:**
- Using feature flags we can expose inputs for Terraform.
- Reduce release coupling. Decouple low-risk operational changes from full plan apply cycles.
- Platform teams own the terraform, developers/managers own the inputs based on the enterprise Feature Management's approval flows.

{{< youtube "TtlhTvoco7Y" >}}

**Resources:**
- [📊 Slides](https://example.com/slides/kubecon-2025)
- [💻 Demo Code](https://github.com/example/resilient-platforms)
- [📝 Blog Post](https://example.com/blog/multi-tenant-platforms)

---

## Speaking

Interested in having me speak at your conference, meetup, or company event? I'm particularly interested in topics around:

{{< cards >}}
{{< card icon="server" title="Platform Engineering" subtitle="Developer platforms, platform team organization, and enabling engineering velocity" >}}
{{< card icon="shield-check" title="Infrastructure Reliability" subtitle="SRE practices, operational patterns, and building systems that degrade gracefully" >}}
{{< card icon="lock-closed" title="Security at Velocity" subtitle="DevSecOps, security as enabling constraints, and compliance in regulated environments" >}}
{{< card icon="light-bulb" title="Making Complexity Understandable" subtitle="System design, architecture patterns, and communicating technical trade-offs" >}}
{{< /cards >}}

**Past audiences:** Engineers, engineering managers, platform teams, SRE teams, and technical leadership.

**Talk formats:** Conference keynotes (30-45 min), technical deep-dives (45-60 min), workshops (half-day or full-day), internal lunch-and-learns.

Feel free to reach out via [email](mailto:hellows@hosh.io) or [LinkedIn](https://linkedin.com/in/hoshsadiq).

