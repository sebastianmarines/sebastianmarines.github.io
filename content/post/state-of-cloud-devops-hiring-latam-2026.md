---
title: "State of Cloud & DevOps Hiring in LATAM: What 8,500+ Job Listings Reveal"
date: 2026-04-14T12:00:00-06:00
summary: "I scraped 8,500+ cloud-adjacent job listings across 14 LATAM countries from four job boards, enriched them with AI, and analyzed the results. Here's what the data says about cloud providers, salaries, toolchains, remote work, company ratings, and the staffing agency problem in the region."
author: "Sebastian Marines"
categories: ["data", "cloud", "career"]
tags: ["data", "cloud", "devops", "career", "latam", "aws", "azure", "gcp"]
---

I have been building a job board focused on cloud and DevOps roles in Latin America. As part of that effort, I wrote scrapers for four major job boards (Glassdoor, Workable, LinkedIn, and Greenhouse), collected thousands of listings across LATAM countries and extracted salaries, certifications, locations, skills, and seniority.

This post is a summary of what the data looks like as of April 2026.

## The dataset

- **8,579 cloud-adjacent job listings** from 4 sources across 14 LATAM countries
- Jobs were filtered by "Core 4 roles (DevOps, SRE, Cloud Engineer, Platform Engineer) **plus** any other role that either lists a cloud provider or an IaC / orchestration tool in its required skills", so a backend role asking for Kubernetes and Terraform is in, a pure frontend role is not.
- Only active (still-open) listings are included.
- Listings were collected between November 2025 and April 2026.
- For each listing I used regular expressions to extract role seniority, place of work (remote/onsite/hybrid), mentions of cloud providers, certifications, required skills, and salary.
- After this first pass, I used Claude Haiku 4.5 to classify jobs into a list of standardized roles, certifications that were mentioned without a clear name, to extract years of experience requirements, english level, and check for "rockstar culture", unpaid trials, vague compensation, excessive requirements, and other red flags.


{{< chartjs id="sourceChart" >}}
new Chart(document.getElementById('sourceChart'), {
    type: 'bar',
    data: {
        labels: ['Glassdoor', 'LinkedIn', 'Workable', 'Greenhouse'],
        datasets: [{
            label: 'Jobs',
            data: [4818, 2629, 1032, 100],
            backgroundColor: ['#818cf8', '#c084fc', '#38bdf8', '#2dd4bf'],
            borderRadius: 6,
        }]
    },
    options: {
        plugins: { legend: { display: false } },
        scales: { y: { beginAtZero: true } }
    }
});
{{< /chartjs >}}

Glassdoor and LinkedIn together account for **87% of the dataset**, with Workable contributing another 12% and Greenhouse a long-tail 1%. Glassdoor skews toward direct employers, while Workable — despite its smaller volume — is by far the agency-heaviest source (more on that below).

## Role distribution

DevOps dominates with **1,706 listings (19.9%)**, followed by Backend (1,303 / 15.2%), Data Engineer (776 / 9.0%), Cloud Engineer (732 / 8.5%), SRE (678 / 7.9%), Fullstack (623 / 7.3%), ML Engineer (414 / 4.8%), Solutions Architect (403 / 4.7%), Platform Engineer (300 / 3.5%), QA (279 / 3.3%), Security (216 / 2.5%), Data Scientist (185 / 2.2%), and **MLOps (148 / 1.7%)**. Note that "backend" and "data engineer" are in the dataset only because they require cloud skills, so the counts reflect backend/data roles where the company expects you to own infrastructure or work closely with cloud technologies.

MLOps is broken out from ML Engineer on purpose: ML Engineer in this dataset is the applied-ML / training / feature-engineering role, while MLOps is the platform-and-serving side — MLflow, SageMaker, Vertex AI, Kubeflow, Triton, Ray, feature stores, GPU cluster management. They hire for different skill stacks: MLOps listings mention AI/ML tooling in 37.2% of postings vs just 16.7% for ML Engineer, consistent with MLOps being the role that actually maintains the ML infrastructure day-to-day.

{{< chartjs id="roleChart" >}}
new Chart(document.getElementById('roleChart'), {
    type: 'bar',
    data: {
        labels: ['DevOps', 'Backend', 'Data Engineer', 'Cloud Engineer', 'SRE', 'Fullstack', 'ML Engineer', 'Solutions Architect', 'Platform Eng.', 'QA', 'Security', 'Data Scientist', 'MLOps'],
        datasets: [{
            data: [1706, 1303, 776, 732, 678, 623, 414, 403, 300, 279, 216, 185, 148],
            backgroundColor: '#38bdf8',
            borderRadius: 4,
        }]
    },
    options: {
        indexAxis: 'y',
        plugins: { legend: { display: false } },
        scales: { x: { beginAtZero: true } }
    }
});
{{< /chartjs >}}

## Location & Language

### Distribution by country

Brazil is firmly #1 with **3,296 listings (38.4% of the dataset)**, more than Mexico (2,296 / 26.8%) and Argentina/Colombia combined. Argentina (614 / 7.2%) and Colombia (574 / 6.7%) are essentially tied for third, followed by Chile (210 / 2.4%), Peru (159 / 1.9%), Costa Rica (105 / 1.2%), and Uruguay (48 / 0.6%). Another ~1,160 listings (13.5%) don't specify a country, these are typically remote-first roles posted without a location requirement.

{{< chartjs id="geoChart" >}}
new Chart(document.getElementById('geoChart'), {
    type: 'bar',
    data: {
        labels: ['Brazil', 'Mexico', 'Argentina', 'Colombia', 'Chile', 'Peru', 'Costa Rica', 'Uruguay'],
        datasets: [{
            data: [3296, 2296, 614, 574, 210, 159, 105, 48],
            backgroundColor: '#38bdf8',
            borderRadius: 4,
        }]
    },
    options: {
        indexAxis: 'y',
        plugins: { legend: { display: false } },
        scales: { x: { beginAtZero: true } }
    }
});
{{< /chartjs >}}

### Listing language

Most listings are written in **English (70.6%)**, with Portuguese (21.3%) and Spanish (8.1%) trailing. Portuguese has grown noticeably as Brazilian coverage expanded — more than one in five listings is now in Portuguese. Still, engineers comfortable reading technical English have a meaningfully larger pool to apply to: even for roles physically based in Brazil or Mexico, the job description itself is often in English.

{{< chartjs id="langPostingChart" width="450px" >}}
new Chart(document.getElementById('langPostingChart'), {
    type: 'doughnut',
    data: {
        labels: ['English (70.6%)', 'Portuguese (21.3%)', 'Spanish (8.1%)'],
        datasets: [{
            data: [6057, 1826, 696],
            backgroundColor: ['#38bdf8', '#4ade80', '#fbbf24'],
            borderWidth: 0,
        }]
    },
    options: {
        cutout: '55%',
        plugins: { legend: { position: 'bottom' } }
    }
});
{{< /chartjs >}}

### English proficiency required

The posting language is one signal, the *required English level for the applicant* is another. Out of 1,643 listings, **79.1% demand advanced or fluent English**. Only 0.9% say "basic" is enough. Effectively, if a listing specifies an English level at all, it's almost certainly going to ask for advanced.

{{< chartjs id="englishLevelChart" width="450px" >}}
new Chart(document.getElementById('englishLevelChart'), {
    type: 'doughnut',
    data: {
        labels: ['Advanced (51.1%)', 'Fluent (28.0%)', 'Intermediate (19.9%)', 'Basic (0.9%)'],
        datasets: [{
            data: [840, 460, 327, 15],
            backgroundColor: ['#2dd4bf', '#38bdf8', '#fbbf24', '#fb7185'],
            borderWidth: 0,
        }]
    },
    options: {
        cutout: '55%',
        plugins: { legend: { position: 'bottom' } }
    }
});
{{< /chartjs >}}



## Remote vs Onsite vs Hybrid

Out of **6,553 listings with a classified work arrangement**, **52.9% are remote**, **27.2% hybrid**, and **19.9% onsite**. Remote is clearly the plurality, with hybrid a solid second — together they account for 80% of the market, leaving onsite-only as the minority arrangement.

{{< chartjs id="remoteChart" width="450px" >}}
new Chart(document.getElementById('remoteChart'), {
    type: 'doughnut',
    data: {
        labels: ['Remote (52.9%)', 'Hybrid (27.2%)', 'Onsite (19.9%)'],
        datasets: [{
            data: [52.9, 27.2, 19.9],
            backgroundColor: ['#2dd4bf', '#fbbf24', '#fb7185'],
            borderWidth: 0,
        }]
    },
    options: {
        cutout: '55%',
        plugins: { legend: { position: 'bottom' } }
    }
});
{{< /chartjs >}}

Remote availability varies widely by country. Costa Rica (62.8%) and Argentina (60.6%) lead; Mexico (39.8%) and Peru (39.9%) sit at the bottom. Brazil — the biggest market — lands at 46.3% remote, notably lower than most LATAM remote discourse assumes. Colombia (55.1%) sits in the middle as a genuinely remote-friendly market.

{{< chartjs id="remoteCountryChart" >}}
new Chart(document.getElementById('remoteCountryChart'), {
    type: 'bar',
    data: {
        labels: ['Costa Rica', 'Argentina', 'Colombia', 'Brazil', 'Chile', 'Peru', 'Mexico'],
        datasets: [{
            label: 'Remote %',
            data: [62.8, 60.6, 55.1, 46.3, 41.0, 39.9, 39.8],
            backgroundColor: '#2dd4bf',
            borderRadius: 4,
        }]
    },
    options: {
        plugins: { legend: { display: false } },
        scales: { y: { beginAtZero: true, max: 100, ticks: { callback: function(v) { return v + '%'; } } } }
    }
});
{{< /chartjs >}}

Remote availability climbs with seniority: only **39.6% of junior roles** are remote vs **61.0% of senior** and **65.8% of staff** positions. Companies trust experienced engineers to work without supervision; juniors are the only tier where hybrid (32.5%) and onsite (27.9%) combined beat remote.

{{< chartjs id="remoteSeniorityChart" >}}
new Chart(document.getElementById('remoteSeniorityChart'), {
    type: 'bar',
    data: {
        labels: ['Junior', 'Mid', 'Senior', 'Lead', 'Management', 'Staff'],
        datasets: [
            { label: 'Remote %', data: [39.6, 48.6, 61.0, 60.1, 53.8, 65.8], backgroundColor: '#2dd4bf', borderRadius: 3 },
            { label: 'Hybrid %', data: [32.5, 28.9, 24.2, 24.5, 26.2, 22.8], backgroundColor: '#fbbf24', borderRadius: 3 },
            { label: 'Onsite %', data: [27.9, 22.6, 14.9, 15.4, 19.9, 11.4], backgroundColor: '#fb7185', borderRadius: 3 },
        ]
    },
    options: {
        plugins: { legend: { position: 'top' } },
        scales: { x: { stacked: true }, y: { stacked: true, ticks: { callback: function(v) { return v + '%'; } } } }
    }
});
{{< /chartjs >}}

## The Cloud providers

### AWS leads, but Azure is right behind

[AWS](https://aws.amazon.com/) leads with **4,640 mentions**, but [Azure](https://azure.microsoft.com/) is closer than most people expect at **3,812**. [GCP](https://cloud.google.com/) trails at 2,380. Counts are not mutually exclusive: listings routinely mention multiple providers.

Multi-cloud is substantial: **1,395 jobs mention all three clouds**, 1,030 want exactly two, and 4,587 (53%) stick to a single provider. More than one in four listings in this dataset expects multi-cloud experience.

{{< chartjs id="cloudChart" >}}
new Chart(document.getElementById('cloudChart'), {
    type: 'bar',
    data: {
        labels: ['AWS', 'Azure', 'GCP'],
        datasets: [{
            data: [4640, 3812, 2380],
            backgroundColor: ['#fbbf24', '#38bdf8', '#4ade80'],
            borderRadius: 6,
            barThickness: 50,
        }]
    },
    options: {
        indexAxis: 'y',
        plugins: { legend: { display: false } },
        scales: { x: { beginAtZero: true } }
    }
});
{{< /chartjs >}}

If we break it down by seniority, something interesting happens: **Azure actually leads AWS in junior roles (45.5% vs 39.3%)**. AWS then pulls ahead by mid-level and keeps extending its lead, at the staff level it hits 69.0% vs Azure's 26.0%. Azure appears to be the more common "first cloud" in enterprise entry-level work, while AWS dominates senior cloud-native engineering.

{{< chartjs id="cloudSeniorityChart" >}}
new Chart(document.getElementById('cloudSeniorityChart'), {
    type: 'bar',
    data: {
        labels: ['Junior', 'Mid', 'Senior', 'Lead', 'Management', 'Staff'],
        datasets: [
            { label: 'AWS %', data: [39.3, 51.8, 59.3, 56.3, 54.6, 69.0], backgroundColor: '#fbbf24', borderRadius: 3 },
            { label: 'Azure %', data: [45.5, 45.4, 42.5, 45.4, 46.9, 26.0], backgroundColor: '#38bdf8', borderRadius: 3 },
            { label: 'GCP %', data: [19.9, 28.2, 25.8, 30.1, 37.0, 26.0], backgroundColor: '#4ade80', borderRadius: 3 },
        ]
    },
    options: {
        plugins: { legend: { position: 'top' } },
        scales: { y: { beginAtZero: true, ticks: { callback: function(v) { return v + '%'; } } } }
    }
});
{{< /chartjs >}}

By country, AWS leads in every major market **except Peru and Costa Rica**; in both, Azure is the most-requested cloud. Mexico — long the most Azure-heavy LATAM market — has flipped to AWS (**1,140 AWS vs 1,075 Azure**), though only narrowly. Peru stays clearly Azure-leaning (73 vs 64), Costa Rica is Azure-heavy (51 vs 36), and Chile now favors AWS (80) ahead of GCP (72) and Azure (70). Brazil, Colombia, Argentina, and Uruguay all show a clear AWS lead.

{{< chartjs id="cloudCountryChart" >}}
new Chart(document.getElementById('cloudCountryChart'), {
    type: 'bar',
    data: {
        labels: ['Brazil', 'Mexico', 'Argentina', 'Colombia', 'Chile', 'Peru', 'Costa Rica'],
        datasets: [
            { label: 'AWS', data: [1915, 1140, 328, 273, 80, 64, 36], backgroundColor: '#fbbf24', borderRadius: 3 },
            { label: 'Azure', data: [1422, 1075, 219, 214, 70, 73, 51], backgroundColor: '#38bdf8', borderRadius: 3 },
            { label: 'GCP', data: [952, 587, 155, 122, 72, 28, 18], backgroundColor: '#4ade80', borderRadius: 3 },
        ]
    },
    options: {
        plugins: { legend: { position: 'top' } },
        scales: { y: { beginAtZero: true } }
    }
});
{{< /chartjs >}}

## Required skills & tools

### Infrastructure as Code

[Terraform](https://www.terraform.io/) absolutely dominates with **2,222 mentions**, over 3x [Ansible](https://www.ansible.com/) (706) and more than 4x [CloudFormation](https://aws.amazon.com/cloudformation/) (516). [Pulumi](https://www.pulumi.com/) (120) edges out [Puppet](https://www.puppet.com/) (110), but both remain niche.

{{< chartjs id="iacChart" >}}
new Chart(document.getElementById('iacChart'), {
    type: 'bar',
    data: {
        labels: ['Terraform', 'Ansible', 'CloudFormation', 'Pulumi', 'Puppet'],
        datasets: [{ data: [2222, 706, 516, 120, 110], backgroundColor: '#2dd4bf', borderRadius: 6 }]
    },
    options: {
        indexAxis: 'y',
        plugins: { legend: { display: false } },
        scales: { x: { beginAtZero: true } }
    }
});
{{< /chartjs >}}

### CI/CD

**[GitHub Actions](https://github.com/features/actions) and [Jenkins](https://www.jenkins.io/) are essentially tied at the top** (945 vs 918). [GitLab CI](https://docs.gitlab.com/ci/) follows at 410. [ArgoCD](https://argo-cd.readthedocs.io/) (203) continues to grow as a GitOps staple. [Azure DevOps](https://azure.microsoft.com/en-us/products/devops) sits lower at 163 — it's widely used at enterprises in Mexico and Peru but shows up less often as an explicit skill requirement than you'd think.

{{< chartjs id="cicdChart" >}}
new Chart(document.getElementById('cicdChart'), {
    type: 'bar',
    data: {
        labels: ['GitHub Actions', 'Jenkins', 'GitLab CI', 'ArgoCD', 'Azure DevOps', 'CircleCI'],
        datasets: [{ data: [945, 918, 410, 203, 163, 60], backgroundColor: '#818cf8', borderRadius: 6 }]
    },
    options: {
        indexAxis: 'y',
        plugins: { legend: { display: false } },
        scales: { x: { beginAtZero: true } }
    }
});
{{< /chartjs >}}

### Monitoring & Observability

The open-source stack wins: **[Grafana](https://grafana.com/) (816) and [Prometheus](https://prometheus.io/) (714)** lead the category. [Datadog](https://www.datadoghq.com/) (519) is the top commercial alternative, with [CloudWatch](https://aws.amazon.com/cloudwatch/) (384) close behind as the AWS-native default. The Prometheus+Grafana pairing is nearly inseparable: **618 jobs mention both**, while only 96 want Prometheus without Grafana and 198 want Grafana without Prometheus.

{{< chartjs id="monChart" >}}
new Chart(document.getElementById('monChart'), {
    type: 'bar',
    data: {
        labels: ['Grafana', 'Prometheus', 'Datadog', 'CloudWatch', 'Splunk', 'New Relic', 'Dynatrace', 'Zabbix'],
        datasets: [{ data: [816, 714, 519, 384, 164, 158, 37, 25], backgroundColor: '#fbbf24', borderRadius: 6 }]
    },
    options: {
        indexAxis: 'y',
        plugins: { legend: { display: false } },
        scales: { x: { beginAtZero: true } }
    }
});
{{< /chartjs >}}

### Containers

[Kubernetes](https://kubernetes.io/) shows up in **2,813 listings (32.8% of the dataset)**, and [Docker](https://www.docker.com/) in 2,497. They usually are mentioned together: **1,757 listings mention both**.

{{< chartjs id="containerChart" >}}
new Chart(document.getElementById('containerChart'), {
    type: 'bar',
    data: {
        labels: ['K8s + Docker', 'K8s only', 'Docker only'],
        datasets: [{ data: [1757, 1056, 740], backgroundColor: ['#38bdf8', '#818cf8', '#c084fc'], borderRadius: 6 }]
    },
    options: {
        plugins: { legend: { display: false } },
        scales: { y: { beginAtZero: true } }
    }
});
{{< /chartjs >}}

Kubernetes scales with seniority: **19.4% of junior roles** mention it, climbing to **49.0% of staff-level** positions. The curve is smoother than it looks in smaller samples — senior to lead sit in the 34–39% range, and the staff spike is a smaller-sample story.

{{< chartjs id="k8sSeniorityChart" >}}
new Chart(document.getElementById('k8sSeniorityChart'), {
    type: 'bar',
    data: {
        labels: ['Junior', 'Mid', 'Senior', 'Lead', 'Management', 'Staff'],
        datasets: [{
            label: '% of roles mentioning Kubernetes',
            data: [19.4, 32.4, 34.5, 38.5, 19.0, 49.0],
            backgroundColor: ['#fbbf24', '#38bdf8', '#2dd4bf', '#818cf8', '#c084fc', '#fb7185'],
            borderRadius: 6,
        }]
    },
    options: {
        plugins: { legend: { display: false } },
        scales: { y: { beginAtZero: true, ticks: { callback: function(v) { return v + '%'; } } } }
    }
});
{{< /chartjs >}}

## Programming languages

**[Python](https://www.python.org/) dominates** at 3,707 mentions; 2.6x [JavaScript](https://developer.mozilla.org/en-US/docs/Web/JavaScript)/[TypeScript](https://www.typescriptlang.org/) (1,404) and 2.8x [Java](https://www.java.com/) (1,309). [Go](https://go.dev/) is a clear fourth at 908. [C#](https://learn.microsoft.com/en-us/dotnet/csharp/)/[.NET](https://dotnet.microsoft.com/) (704) holds up on the strength of Azure enterprise work, and [Bash](https://www.gnu.org/software/bash/)/Shell (333) is a baseline expectation for anyone touching infrastructure. But [Rust](https://www.rust-lang.org/) remains a tiny niche in the LATAM cloud/DevOps job market, only 75 listings ask for it.

{{< chartjs id="langChart" >}}
new Chart(document.getElementById('langChart'), {
    type: 'bar',
    data: {
        labels: ['Python', 'JS/TypeScript', 'Java', 'Go', 'C#/.NET', 'Bash/Shell', 'Rust', 'Ruby', 'PHP'],
        datasets: [{
            data: [3707, 1404, 1309, 908, 704, 333, 75, 19, 18],
            backgroundColor: ['#2dd4bf','#fbbf24','#fb7185','#38bdf8','#818cf8','#64748b','#f97316','#fb7185','#c084fc'],
            borderRadius: 4,
        }]
    },
    options: {
        indexAxis: 'y',
        plugins: { legend: { display: false } },
        scales: { x: { beginAtZero: true } }
    }
});
{{< /chartjs >}}

## Databases

[PostgreSQL](https://www.postgresql.org/) leads at 951 mentions, followed by [MongoDB](https://www.mongodb.com/) (563), [MySQL](https://www.mysql.com/) (536), [Redis](https://redis.io/) (365), and [DynamoDB](https://aws.amazon.com/dynamodb/) (340). [SQL Server](https://www.microsoft.com/en-us/sql-server) (59) is surprisingly low given how many enterprise Azure roles are in the dataset, likely because SQL Server expertise often sits on DBA job descriptions, not DevOps ones. [Elasticsearch](https://www.elastic.co/elasticsearch) (268) rounds out the observability-adjacent data layer.

{{< chartjs id="dbChart" >}}
new Chart(document.getElementById('dbChart'), {
    type: 'bar',
    data: {
        labels: ['PostgreSQL', 'MongoDB', 'MySQL', 'Redis', 'DynamoDB', 'Elasticsearch', 'SQL Server'],
        datasets: [{ data: [951, 563, 536, 365, 340, 268, 59], backgroundColor: '#c084fc', borderRadius: 6 }]
    },
    options: {
        indexAxis: 'y',
        plugins: { legend: { display: false } },
        scales: { x: { beginAtZero: true } }
    }
});
{{< /chartjs >}}

## Seniority and the scarcity of junior roles

**Junior positions are only 2.2%** of the market (191 out of 8,579). Mid-level roles dominate at **61.0%**, senior is 26.3%, and lead/management/staff together make up 10.5%. LATAM Cloud/DevOps employers expect hires to show up pre-trained.

I find this dynamic very interesting, and I will continue to track seniority distribution over time as the market matures. My hypothesis is that 1) DevOps/Cloud jobs are more senior-heavy than the general market because of the technical complexity, and 2) AI is affecting the junior end of the market more than the senior end. Comparing this snapshot to the previous one (where junior was 2.1%), the share has barely moved even as the dataset grew by ~70%, which is consistent with both hypotheses — but it's still a short time window, so treat it as a snapshot rather than a trend.

{{< chartjs id="seniorityChart" >}}
new Chart(document.getElementById('seniorityChart'), {
    type: 'bar',
    data: {
        labels: ['Mid-level', 'Senior', 'Lead', 'Management', 'Junior', 'Staff'],
        datasets: [{
            data: [5234, 2259, 522, 273, 191, 100],
            backgroundColor: ['#38bdf8', '#2dd4bf', '#818cf8', '#c084fc', '#fbbf24', '#fb7185'],
            borderRadius: 6,
        }]
    },
    options: {
        indexAxis: 'y',
        plugins: { legend: { display: false } },
        scales: { x: { beginAtZero: true } }
    }
});
{{< /chartjs >}}

### Experience Expectations

Across 3,050 listings that spelled out years-of-experience requirements, the average minimum is **4.6 years**. Only **380 listings (12.5%)** ask for two years or less. The bulk of the market sits in the **3-5 year** bucket.

{{< chartjs id="yoeChart" >}}
new Chart(document.getElementById('yoeChart'), {
    type: 'bar',
    data: {
        labels: ['0', '1-2', '3-5', '6-9', '10+'],
        datasets: [{
            label: 'Listings (min years required)',
            data: [23, 357, 2049, 515, 106],
            backgroundColor: ['#4ade80','#fbbf24','#38bdf8','#c084fc','#fb7185'],
            borderRadius: 4,
        }]
    },
    options: {
        plugins: { legend: { display: false } },
        scales: { y: { beginAtZero: true } }
    }
});
{{< /chartjs >}}

### Education requirements

Most listings don't mention education at all, but when they do, the picture is clear. Out of **2,299 listings** that spelled out an education requirement, **67.8% ask for a bachelor's degree**, 15.2% explicitly state no degree is required (or "or equivalent experience"), 9.0% ask for a master's, 7.6% for an associate's / técnico superior, and fewer than 1% mention bootcamp-level credentials. PhD requirements in the cloud/DevOps space are effectively zero.

{{< chartjs id="educationLevelChart" width="450px" >}}
new Chart(document.getElementById('educationLevelChart'), {
    type: 'doughnut',
    data: {
        labels: ['Bachelor (67.8%)', 'No degree required (15.2%)', "Master's (9.0%)", "Associate / Técnico (7.6%)", 'PhD (0.3%)'],
        datasets: [{
            data: [1559, 350, 208, 175, 6],
            backgroundColor: ['#38bdf8', '#2dd4bf', '#c084fc', '#fbbf24', '#fb7185'],
            borderWidth: 0,
        }]
    },
    options: {
        cutout: '55%',
        plugins: { legend: { position: 'bottom' } }
    }
});
{{< /chartjs >}}

When a field of study is named, it's overwhelmingly Computer Science (818 listings), followed by generic Engineering (155), IT (64), Software Engineering (33), and Information Systems (30). Data Science (26), Mathematics, Statistics, and Electrical Engineering together add up to fewer than 55 listings.

The degree-required share doesn't move linearly with seniority. Junior roles ask for a degree more often (33.5%) than mid-level (20.8%) or senior (19.3%) roles — employers use credentials as a filter when they can't yet judge candidates on experience. Management bounces back to 29.3%, likely a credentialism bias for people-management roles.

{{< chartjs id="educationSeniorityChart" >}}
new Chart(document.getElementById('educationSeniorityChart'), {
    type: 'bar',
    data: {
        labels: ['Junior', 'Mid', 'Senior', 'Lead', 'Management', 'Staff'],
        datasets: [{
            label: '% of listings requiring a degree',
            data: [33.5, 20.8, 19.3, 14.4, 29.3, 31.0],
            backgroundColor: '#38bdf8',
            borderRadius: 4,
        }]
    },
    options: {
        plugins: { legend: { display: false } },
        scales: { y: { beginAtZero: true, max: 50, ticks: { callback: function(v) { return v + '%'; } } } }
    }
});
{{< /chartjs >}}

By country, Mexico asks for a degree most often (25.7% of listings), followed by Brazil (22.0%), Costa Rica (18.1%), Argentina (12.7%), Colombia (12.5%), Peru (10.7%), and Chile (8.1%). Chile remains the most credentialism-free market in the region for cloud/DevOps work.

{{< chartjs id="educationCountryChart" >}}
new Chart(document.getElementById('educationCountryChart'), {
    type: 'bar',
    data: {
        labels: ['Mexico', 'Brazil', 'Costa Rica', 'Argentina', 'Colombia', 'Peru', 'Chile'],
        datasets: [{
            label: '% of listings requiring a degree',
            data: [25.7, 22.0, 18.1, 12.7, 12.5, 10.7, 8.1],
            backgroundColor: '#38bdf8',
            borderRadius: 4,
        }]
    },
    options: {
        plugins: { legend: { display: false } },
        scales: { y: { beginAtZero: true, max: 35, ticks: { callback: function(v) { return v + '%'; } } } }
    }
});
{{< /chartjs >}}

## Salaries

Explicit USD salary figures are rare, only **166 listings** had mentions of the salary range. While the sample is small, the data is still interesting: the average minimum salary is $36,500 and the average maximum is $55,000.

By seniority, junior roles cluster around $16K–$28K, mid-level in the $27K–$40K band, senior reaches $62K–$87K, and leads go $63K–$124K. Management comes in below lead level.

{{< chartjs id="salaryUsdChart" >}}
new Chart(document.getElementById('salaryUsdChart'), {
    type: 'bar',
    data: {
        labels: ['Junior (n=11)', 'Mid (n=102)', 'Senior (n=36)', 'Lead (n=8)', 'Management (n=9)'],
        datasets: [
            { label: 'Avg Min', data: [15815, 26548, 61946, 62879, 50784], backgroundColor: '#64748b', borderRadius: 3 },
            { label: 'Avg Max', data: [28246, 39523, 86944, 124425, 74631], backgroundColor: '#fbbf24', borderRadius: 3 },
        ]
    },
    options: {
        plugins: { legend: { position: 'top' } },
        scales: { y: { beginAtZero: true, ticks: { callback: function(v) { return '$' + (v/1000).toFixed(0) + 'K'; } } } }
    }
});
{{< /chartjs >}}

## Certifications

**6.6% of listings mention specific certifications.** That's noticeably lower than the earlier snapshot (16.9%) — the scraper now covers a wider pool of general cloud/DevOps postings, and most listings simply don't name a credential. Among the ones that do, [Terraform Associate](https://www.hashicorp.com/certification/terraform-associate) leads with 68 mentions, followed by [Azure Administrator Associate](https://learn.microsoft.com/en-us/credentials/certifications/azure-administrator/) (AZ-104, combined with its legacy label) at 65. [AWS Solutions Architect Associate](https://aws.amazon.com/certification/certified-solutions-architect-associate/) and [CKA](https://www.cncf.io/training/certification/cka/) are tied at 60. [Azure Solutions Architect](https://learn.microsoft.com/en-us/credentials/certifications/azure-solutions-architect/) comes in at 54, well ahead of [AWS Solutions Architect Professional](https://aws.amazon.com/certification/certified-solutions-architect-professional/) (14).

{{< chartjs id="certChart" >}}
new Chart(document.getElementById('certChart'), {
    type: 'bar',
    data: {
        labels: ['Terraform Assoc.', 'Azure Admin Assoc. (AZ-104)', 'AWS SA Assoc.', 'CKA', 'Azure Sol. Arch.', 'AWS SA Pro'],
        datasets: [{
            data: [68, 65, 60, 60, 54, 14],
            backgroundColor: ['#2dd4bf','#38bdf8','#fbbf24','#fb7185','#38bdf8','#fbbf24'],
            borderRadius: 4,
        }]
    },
    options: {
        indexAxis: 'y',
        plugins: { legend: { display: false } },
        scales: { x: { beginAtZero: true } }
    }
});
{{< /chartjs >}}

## Staffing agencies vs direct employers

### Top hiring companies

Outsourcing giants lead: **EPAM Systems (339 listings)**, Capgemini (107), Accenture (46), and Infosys (41) are here in force. LATAM-native staffing firms are a major presence too — DaCodes (83), Dev.Pro (63), Jalasoft (55), Workana (49), Pavago (44), Maxana (41), and Activate Talent (38). Brazilian IT-services players like CI&T (70) and Serasa Experian (44) round out the top of the list.

Direct employers in the top 14 are largely product companies and a few banks (Sezzle, Serasa Experian, EY); no hyperscaler makes the top 14 this time — the cut is heavier on outsourcing and staffing.

{{< chartjs id="companyChart" >}}
new Chart(document.getElementById('companyChart'), {
    type: 'bar',
    data: {
        labels: ['EPAM', 'Capgemini', 'DaCodes', 'CI&T', 'Dev.Pro', 'Jalasoft', 'Sezzle', 'Workana', 'Accenture', 'EY', 'Pavago', 'Serasa Experian', 'Infosys', 'Maxana'],
        datasets: [{
            data: [339, 107, 83, 70, 63, 55, 53, 49, 46, 45, 44, 44, 41, 41],
            backgroundColor: '#38bdf8',
            borderRadius: 3,
        }]
    },
    options: {
        indexAxis: 'y',
        plugins: { legend: { display: false } },
        scales: { x: { beginAtZero: true } }
    }
});
{{< /chartjs >}}

### Staffing agencies

**8.2% of listings (706 jobs) come from staffing agencies.** The country-level breakdown: **Argentina (17.4%), Peru (15.7%), Costa Rica (12.4%), Colombia (12.4%), and Chile (11.9%)** have the heaviest agency presence; Brazil (5.4%) and Mexico (6.7%) are much cleaner markets. By source, Workable is by far the agency-heaviest — **30.8% of Workable listings are agency-posted**, vs about 5% on Glassdoor and LinkedIn.

{{< chartjs id="agencyCountryChart" >}}
new Chart(document.getElementById('agencyCountryChart'), {
    type: 'bar',
    data: {
        labels: ['Argentina', 'Peru', 'Costa Rica', 'Colombia', 'Chile', 'Mexico', 'Brazil'],
        datasets: [{
            label: 'Agency %',
            data: [17.4, 15.7, 12.4, 12.4, 11.9, 6.7, 5.4],
            backgroundColor: '#fb7185',
            borderRadius: 6,
        }]
    },
    options: {
        plugins: { legend: { display: false } },
        scales: { y: { beginAtZero: true, max: 25, ticks: { callback: function(v) { return v + '%'; } } } }
    }
});
{{< /chartjs >}}

### Agencies still have worse Glassdoor ratings

Agencies are a major presence in the LATAM Cloud/DevOps market, and they still don't have a great reputation. Among listings with Glassdoor ratings (4,245 direct employers vs 216 staffing firms), agencies score worse in every dimension, with the biggest gaps in **compensation (3.49 vs 3.13, -0.36)** and **culture (3.65 vs 3.31, -0.34)**. The overall-rating gap is now small (3.75 vs 3.72), but the sub-dimensions that matter for day-to-day work — comp, culture, work/life — all favor direct employers.

{{< chartjs id="agencyRatingsChart" >}}
new Chart(document.getElementById('agencyRatingsChart'), {
    type: 'bar',
    data: {
        labels: ['Overall', 'Work/Life', 'Compensation', 'Culture', 'Career Opps'],
        datasets: [
            { label: 'Direct Employer', data: [3.75, 3.66, 3.49, 3.65, 3.49], backgroundColor: '#64748b', borderRadius: 3 },
            { label: 'Staffing Agency',  data: [3.72, 3.42, 3.13, 3.31, 3.24], backgroundColor: '#fb7185', borderRadius: 3 },
        ]
    },
    options: {
        plugins: { legend: { position: 'top' } },
        scales: { y: { beginAtZero: true, max: 5 } }
    }
});
{{< /chartjs >}}

## Takeaways

If you're a cloud/DevOps engineer in LATAM:

- **Learn Terraform.** It's the dominant IaC tool (2,222 mentions, 3x anything else), and Terraform Associate is the single most-requested certification, just ahead of AZ-104 and AWS SA Associate.
- **Don't sleep on Azure.** Azure is closer to AWS than most people think (3,812 vs 4,640 total; Azure is actually ahead in Peru and Costa Rica, and in junior-level roles across LATAM). Mexico — historically the most Azure-heavy market — just flipped to AWS by a narrow margin. In enterprise-heavy markets, Azure is still the pragmatic first cloud to learn.
- **Python first, everything else second.** Python appears in 3,707 listings, 2.6x JS/TS, 2.8x Java, 4.1x Go, 49x Rust.
- **Kubernetes is expected.** 32.8% of roles overall mention K8s, rising to 49% at staff level. Get comfortable with it early.
- **Watch out for agencies.** 8.2% of listings are from staffing agencies, and they rate meaningfully worse on Glassdoor — compensation especially (3.13 vs 3.49). On Workable, roughly one in three listings is agency-posted; Argentina (17.4%), Peru (15.7%), Costa Rica, Colombia, and Chile are the agency-heavy markets. Always verify you're talking to the actual employer.
- **Junior roles are scarce.** Only 2.2% of listings are entry-level, and most ask for 1–2 years of experience anyway. Certifications (Terraform Associate, AZ-104, AWS SA Associate, CKA) are one of the few signals that will move you past resume filters.
- **English matters, but Portuguese is a real chunk.** 70.6% of listings are in English, 21.3% in Portuguese, 8.1% in Spanish. Even roles based in Brazil or Mexico often publish in English first. Being comfortable in technical English roughly triples your addressable pool.
- **Remote is the plurality — hybrid is a strong second.** 53% remote, 27% hybrid, 20% onsite. If you want remote, Costa Rica (63%), Argentina (61%), and Colombia (55%) lead; Mexico, Peru, and Chile (all ~40%) skew hybrid/onsite.
- **MLOps is emerging as its own category.** ML Engineer (414) and MLOps (148) together make up ~6.6% of cloud-adjacent listings — and MLOps roles show a dramatically higher rate of AI/ML keyword mentions (37.2%) than ML Engineer itself (16.7%). If you're moving into ML infrastructure, the platform-side roles increasingly hire for skills that look more like DevOps with model-serving tooling (MLflow, SageMaker, Vertex AI, Kubeflow, Triton, Ray) than for research ML.
