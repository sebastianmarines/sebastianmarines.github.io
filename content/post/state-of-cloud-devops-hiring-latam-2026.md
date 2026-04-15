---
title: "State of Cloud & DevOps Hiring in LATAM: What 5,000+ Job Listings Reveal"
date: 2026-04-14T12:00:00-06:00
summary: "I scraped 5,000+ cloud-adjacent job listings across 14 LATAM countries from four job boards, enriched them with AI, and analyzed the results. Here's what the data says about cloud providers, salaries, toolchains, remote work, company ratings, and the staffing agency problem in the region."
author: "Sebastian Marines"
categories: ["data", "cloud", "career"]
tags: ["data", "cloud", "devops", "career", "latam", "aws", "azure", "gcp"]
draft: true
---

I have been building a job board focused on cloud and DevOps roles in Latin America. As part of that effort, I wrote scrapers for four major job boards (Glassdoor, Workable, LinkedIn, and Greenhouse), collected thousands of listings across LATAM countries and extracted salaries, certifications, locations, skills, and seniority.

This post is a summary of what the data looks like as of April 2026.

## The dataset

- **5,027 cloud-adjacent job listings** from 4 sources across 14 LATAM countries
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
            data: [2070, 1982, 878, 97],
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

Glassdoor and LinkedIn are the biggest sources, followed by Workable and Greenhouse. Glassdoor skews more towards direct employers, while Workable has a much higher share of staffing agencies.

## Role distribution

DevOps dominates with **1,292 listings (25.7%)**, followed by backend (718), Cloud Engineer (525), SRE (434), and data engineer (391). Note that "backend" and "data engineer" are in the dataset only because they require cloud skills, so the counts reflect backend/data roles where the company expects you to own infrastructure or work closely with cloud technologies.

{{< chartjs id="roleChart" >}}
new Chart(document.getElementById('roleChart'), {
    type: 'bar',
    data: {
        labels: ['DevOps', 'Backend', 'Cloud Engineer', 'SRE', 'Data Engineer', 'Fullstack', 'Solutions Architect', 'Platform Eng.', 'QA', 'ML Engineer', 'Security'],
        datasets: [{
            data: [1292, 718, 525, 434, 391, 374, 245, 214, 167, 129, 97],
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

Brazil is firmly #1 with **1,822 listings (36.2% of the dataset)**, more than Mexico (1,423) and Argentina/Colombia combined. Argentina (451) and Colombia (435) are essentially tied for third. Chile, Peru, and Costa Rica complete the top seven. The remaining ~450 jobs don't specify a country, these are typically remote roles posted without a location requirement.

{{< chartjs id="geoChart" >}}
new Chart(document.getElementById('geoChart'), {
    type: 'bar',
    data: {
        labels: ['Brazil', 'Mexico', 'Argentina', 'Colombia', 'Chile', 'Peru', 'Costa Rica', 'Uruguay'],
        datasets: [{
            data: [1822, 1423, 451, 435, 152, 115, 79, 40],
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

Most listings are written in **English (77.5%)**, with Portuguese (15.0%) and Spanish (7.6%) trailing. Engineers comfortable reading technical English have a meaningfully larger pool to apply to, even for roles physically based in Brazil or Mexico, the job description itself often isn't in Portuguese or Spanish.

{{< chartjs id="langPostingChart" width="450px" >}}
new Chart(document.getElementById('langPostingChart'), {
    type: 'doughnut',
    data: {
        labels: ['English (77.5%)', 'Portuguese (15.0%)', 'Spanish (7.6%)'],
        datasets: [{
            data: [3894, 753, 380],
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

The posting language is one signal, the *required English level for the applicant* is another. Out of 1,051 listings, **79.8% demand advanced or fluent English**. Only 0.8% say "basic" is enough. Effectively, if a listing specifies an English level at all, it's almost certainly going to ask for advanced.

{{< chartjs id="englishLevelChart" width="450px" >}}
new Chart(document.getElementById('englishLevelChart'), {
    type: 'doughnut',
    data: {
        labels: ['Advanced (53.1%)', 'Fluent (26.7%)', 'Intermediate (19.4%)', 'Basic (0.8%)'],
        datasets: [{
            data: [558, 281, 204, 8],
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

Out of **3,924 listings with a classified work arrangement**, **55.6% are remote**, **23.8% onsite**, and **20.6% hybrid**. Remote is still the plurality — and it ticked up slightly from the earlier snapshot — but nearly half of listings still require some physical presence.

{{< chartjs id="remoteChart" width="450px" >}}
new Chart(document.getElementById('remoteChart'), {
    type: 'doughnut',
    data: {
        labels: ['Remote (55.6%)', 'Onsite (23.8%)', 'Hybrid (20.6%)'],
        datasets: [{
            data: [55.6, 23.8, 20.6],
            backgroundColor: ['#2dd4bf', '#fb7185', '#fbbf24'],
            borderWidth: 0,
        }]
    },
    options: {
        cutout: '55%',
        plugins: { legend: { position: 'bottom' } }
    }
});
{{< /chartjs >}}

Remote availability varies widely by country. Costa Rica (67.1%) and Argentina (65.6%) lead; Chile (42.1%) and Peru (43.4%) sit at the bottom. Mexico and Brazil — the two largest markets — are both stuck right around **49% remote**, which is notably lower than most LATAM remote discourse assumes.

{{< chartjs id="remoteCountryChart" >}}
new Chart(document.getElementById('remoteCountryChart'), {
    type: 'bar',
    data: {
        labels: ['Costa Rica', 'Argentina', 'Colombia', 'Brazil', 'Mexico', 'Peru', 'Chile'],
        datasets: [{
            label: 'Remote %',
            data: [67.1, 65.6, 55.7, 49.6, 48.8, 43.4, 42.1],
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

Remote availability also climbs with seniority: only **40.7% of junior roles** are remote vs **63.9% of senior** and **62.8% of lead** positions. Companies trust experienced engineers to work without supervision; juniors are still expected in the office a third of the time.

{{< chartjs id="remoteSeniorityChart" >}}
new Chart(document.getElementById('remoteSeniorityChart'), {
    type: 'bar',
    data: {
        labels: ['Junior', 'Mid', 'Senior', 'Lead', 'Management', 'Staff'],
        datasets: [
            { label: 'Remote %', data: [40.7, 51.3, 63.9, 62.8, 57.7, 61.8], backgroundColor: '#2dd4bf', borderRadius: 3 },
            { label: 'Hybrid %', data: [23.1, 22.0, 18.6, 19.2, 15.4, 17.6], backgroundColor: '#fbbf24', borderRadius: 3 },
            { label: 'Onsite %', data: [36.3, 26.7, 17.6, 18.0, 26.9, 20.6], backgroundColor: '#fb7185', borderRadius: 3 },
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

AWS leads with **2,576 mentions**, but Azure is closer than most people expect at **2,235**. GCP trails at 1,051. Counts are not mutually exclusive: listings routinely mention multiple providers.

Multi-cloud is substantial: **732 jobs mention all three clouds**, 583 want exactly two, and 2,698 (54%) stick to a single provider. Close to one in four listings in this dataset expects multi-cloud experience.

{{< chartjs id="cloudChart" >}}
new Chart(document.getElementById('cloudChart'), {
    type: 'bar',
    data: {
        labels: ['AWS', 'Azure', 'GCP'],
        datasets: [{
            data: [2576, 2235, 1249],
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

If we breake it down by seniority, something interesting happens: **Azure actually leads AWS in junior roles (47.1% vs 37.5%)**. AWS then pulls ahead by mid-level and keeps extending its lead, at the staff level it hits 73.3% vs Azure's 31.1%. Azure appears to be the more common "first cloud" in enterprise entry-level work, while AWS dominates senior cloud-native engineering.

{{< chartjs id="cloudSeniorityChart" >}}
new Chart(document.getElementById('cloudSeniorityChart'), {
    type: 'bar',
    data: {
        labels: ['Junior', 'Mid', 'Senior', 'Lead', 'Management', 'Staff'],
        datasets: [
            { label: 'AWS %', data: [37.5, 48.0, 59.1, 51.9, 49.6, 73.3], backgroundColor: '#fbbf24', borderRadius: 3 },
            { label: 'Azure %', data: [47.1, 44.5, 43.1, 50.0, 46.2, 31.1], backgroundColor: '#38bdf8', borderRadius: 3 },
            { label: 'GCP %', data: [15.4, 24.9, 24.1, 28.8, 31.1, 17.8], backgroundColor: '#4ade80', borderRadius: 3 },
        ]
    },
    options: {
        plugins: { legend: { position: 'top' } },
        scales: { y: { beginAtZero: true, ticks: { callback: function(v) { return v + '%'; } } } }
    }
});
{{< /chartjs >}}

By country, AWS leads in every major market **except Mexico, Peru, and Costa Rica**; in all three, Azure is the most-requested cloud. Mexico is the most Azure-heavy (696 vs 661 AWS), Peru is close (58 vs 51), and Costa Rica is a small sample but also Azure-heavy (40 vs 28). Brazil, Colombia, Argentina, and Chile all show a clear AWS lead.

{{< chartjs id="cloudCountryChart" >}}
new Chart(document.getElementById('cloudCountryChart'), {
    type: 'bar',
    data: {
        labels: ['Brazil', 'Mexico', 'Argentina', 'Colombia', 'Chile', 'Peru', 'Costa Rica'],
        datasets: [
            { label: 'AWS', data: [1024, 661, 238, 203, 59, 51, 28], backgroundColor: '#fbbf24', borderRadius: 3 },
            { label: 'Azure', data: [751, 696, 157, 163, 46, 58, 40], backgroundColor: '#38bdf8', borderRadius: 3 },
            { label: 'GCP', data: [464, 327, 107, 88, 43, 22, 16], backgroundColor: '#4ade80', borderRadius: 3 },
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

Terraform absolutely dominates with **1,579 mentions**, over 3x Ansible (501) and more than 4x CloudFormation (377). Puppet (84) and Pulumi (72) remain niche.

{{< chartjs id="iacChart" >}}
new Chart(document.getElementById('iacChart'), {
    type: 'bar',
    data: {
        labels: ['Terraform', 'Ansible', 'CloudFormation', 'Puppet', 'Pulumi'],
        datasets: [{ data: [1579, 501, 377, 84, 72], backgroundColor: '#2dd4bf', borderRadius: 6 }]
    },
    options: {
        indexAxis: 'y',
        plugins: { legend: { display: false } },
        scales: { x: { beginAtZero: true } }
    }
});
{{< /chartjs >}}

### CI/CD

**GitHub Actions and Jenkins are tied at the top** (672 vs 670). GitLab CI follows at 304. ArgoCD (152) continues to grow as a GitOps staple. Azure DevOps sits lower at 134, it's widely used at enterprises in Mexico/Peru but shows up less often as an explicit skill requirement than you'd think.

{{< chartjs id="cicdChart" >}}
new Chart(document.getElementById('cicdChart'), {
    type: 'bar',
    data: {
        labels: ['GitHub Actions', 'Jenkins', 'GitLab CI', 'ArgoCD', 'Azure DevOps', 'CircleCI'],
        datasets: [{ data: [672, 670, 304, 152, 134, 39], backgroundColor: '#818cf8', borderRadius: 6 }]
    },
    options: {
        indexAxis: 'y',
        plugins: { legend: { display: false } },
        scales: { x: { beginAtZero: true } }
    }
});
{{< /chartjs >}}

### Monitoring & Observability

The open-source stack wins: **Grafana (533) and Prometheus (498)** lead the category. Datadog (333) is the top commercial alternative, with CloudWatch (249) close behind as the AWS-native default. The Prometheus+Grafana pairing is nearly inseparable: **422 jobs mention both**, while only 76 want Prometheus without Grafana and 111 want Grafana without Prometheus.

{{< chartjs id="monChart" >}}
new Chart(document.getElementById('monChart'), {
    type: 'bar',
    data: {
        labels: ['Grafana', 'Prometheus', 'Datadog', 'CloudWatch', 'New Relic', 'Splunk', 'Dynatrace', 'Zabbix'],
        datasets: [{ data: [533, 498, 333, 249, 104, 101, 31, 17], backgroundColor: '#fbbf24', borderRadius: 6 }]
    },
    options: {
        indexAxis: 'y',
        plugins: { legend: { display: false } },
        scales: { x: { beginAtZero: true } }
    }
});
{{< /chartjs >}}

### Containers

Kubernetes shows up in **1,853 listings (36.9% of the dataset)**, and Docker in 1,549. They usually are mentioned together: **1,151 listings mention both**.

{{< chartjs id="containerChart" >}}
new Chart(document.getElementById('containerChart'), {
    type: 'bar',
    data: {
        labels: ['K8s + Docker', 'K8s only', 'Docker only'],
        datasets: [{ data: [1151, 702, 398], backgroundColor: ['#38bdf8', '#818cf8', '#c084fc'], borderRadius: 6 }]
    },
    options: {
        plugins: { legend: { display: false } },
        scales: { y: { beginAtZero: true } }
    }
});
{{< /chartjs >}}

Kubernetes scales sharply with seniority: **22.1% of junior roles** mention it, climbing to **62.2% of staff-level** positions.

{{< chartjs id="k8sSeniorityChart" >}}
new Chart(document.getElementById('k8sSeniorityChart'), {
    type: 'bar',
    data: {
        labels: ['Junior', 'Mid', 'Senior', 'Lead', 'Management', 'Staff'],
        datasets: [{
            label: '% of roles mentioning Kubernetes',
            data: [22.1, 36.6, 37.3, 44.0, 23.5, 62.2],
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

**Python dominates** at 2,034 mentions; 2.5x JavaScript/TypeScript (814) and nearly 3x Java (765). Go is a clear fourth at 551. C#/.NET (436) holds up on the strength of Azure enterprise work, and Bash/Shell (263) is a baseline expectation for anyone touching infrastructure. But Rust remains a tiny niche in the LATAM cloud/DevOps job market, only 38 listings ask for it.

{{< chartjs id="langChart" >}}
new Chart(document.getElementById('langChart'), {
    type: 'bar',
    data: {
        labels: ['Python', 'JS/TypeScript', 'Java', 'Go', 'C#/.NET', 'Bash/Shell', 'Rust', 'Ruby', 'PHP'],
        datasets: [{
            data: [2034, 814, 765, 551, 436, 263, 38, 18, 12],
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

PostgreSQL leads at 566 mentions, followed by MongoDB (348), MySQL (335), Redis (216), and DynamoDB (203). SQL Server (45) is surprisingly low given how many enterprise Azure roles are in the dataset, likely because SQL Server expertise often sits on DBA job descriptions, not DevOps ones.

{{< chartjs id="dbChart" >}}
new Chart(document.getElementById('dbChart'), {
    type: 'bar',
    data: {
        labels: ['PostgreSQL', 'MongoDB', 'MySQL', 'Redis', 'DynamoDB', 'Elasticsearch', 'SQL Server'],
        datasets: [{ data: [566, 348, 335, 216, 203, 166, 45], backgroundColor: '#c084fc', borderRadius: 6 }]
    },
    options: {
        indexAxis: 'y',
        plugins: { legend: { display: false } },
        scales: { x: { beginAtZero: true } }
    }
});
{{< /chartjs >}}

## Seniority and the scarcity of junior roles

**Junior positions are only 2.1%** of the market (104 out of 5,027). Mid-level roles dominate at **61.6%**, senior is 26.8%, and lead/management/staff together make up 9.6%. LATAM Cloud/DevOps employers expect hires to show up pre-trained. 

I find this dynamic very interesting, and I will continue to track seniority distribution over time as the market matures. My hypothesis is that 1) DevOps/Cloud jobs are more senior-heavy than the general market because of the technical complexity, and 2) AI is affecting the junior end of the market more than the senior end. But the current dataset is not yet large enough to draw strong conclusions about trends, so for now it's just a snapshot of a very senior-skewed market.

{{< chartjs id="seniorityChart" >}}
new Chart(document.getElementById('seniorityChart'), {
    type: 'bar',
    data: {
        labels: ['Mid-level', 'Senior', 'Lead', 'Management', 'Junior', 'Staff'],
        datasets: [{
            data: [3095, 1348, 316, 119, 104, 45],
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

Across 1,950 listings that spelled out years-of-experience requirements, the average requirement is **4.6 years**. Only **218 listings (11%)** ask for two years or less . The bulk of the market sits in the **3-5 year** bucket.

{{< chartjs id="yoeChart" >}}
new Chart(document.getElementById('yoeChart'), {
    type: 'bar',
    data: {
        labels: ['0', '1-2', '3-5', '6-9', '10+'],
        datasets: [{
            label: 'Listings (min years required)',
            data: [9, 209, 1321, 340, 71],
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


## Salaries

Explicit USD salary figures are rare, only **103 listings** had mentions of the salary range. While the sample is small, the data is still interesting: the average minimum salary is $28,000 and the average maximum is $50,000.

By seniority, junior roles cluster around $17K–$29K, mid-level in the $27K–$40K band, senior reaches $58K–$89K, and leads go $63K–$124K. Management and staff come in slightly below lead level.

{{< chartjs id="salaryUsdChart" >}}
new Chart(document.getElementById('salaryUsdChart'), {
    type: 'bar',
    data: {
        labels: ['Junior (n=7)', 'Mid (n=54)', 'Senior (n=26)', 'Lead (n=8)', 'Management (n=8)'],
        datasets: [
            { label: 'Avg Min', data: [16658, 26918, 57531, 62879, 53150], backgroundColor: '#64748b', borderRadius: 3 },
            { label: 'Avg Max', data: [29343, 40198, 89180, 124425, 78650], backgroundColor: '#fbbf24', borderRadius: 3 },
        ]
    },
    options: {
        plugins: { legend: { position: 'top' } },
        scales: { y: { beginAtZero: true, ticks: { callback: function(v) { return '$' + (v/1000).toFixed(0) + 'K'; } } } }
    }
});
{{< /chartjs >}}

## Certifications

**16.9% of listings mention specific certifications.** AWS Solutions Architect Associate leads at 113 mentions, followed by Terraform Associate (98). Azure Administrator Associate comes in third at **64**, ahead of CKA (54), AWS Solutions Architect Professional (54), and Azure Solutions Architect Expert (43).

{{< chartjs id="certChart" >}}
new Chart(document.getElementById('certChart'), {
    type: 'bar',
    data: {
        labels: ['AWS SA Assoc.', 'Terraform Assoc.', 'Azure Admin Assoc. (AZ-104)', 'CKA', 'AWS SA Pro', 'Azure Sol. Arch. Expert'],
        datasets: [{
            data: [113, 98, 64, 54, 54, 43],
            backgroundColor: ['#fbbf24','#2dd4bf','#38bdf8','#fb7185','#fbbf24','#38bdf8'],
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

Outsourcing giants lead:  **EPAM Systems (155 listings)**, Capgemini (75), and Infosys (34) are here in force. So are LATAM-native staffing firms: DaCodes (63), Dev.Pro (50), and Workana (43).

Direct employers are mostly a mix of local tech companies and global cloud consultancies. The only hyperscalers in the top 14 are AWS (27) and Google Cloud (11), Microsoft Azure doesn't even make the cut.

{{< chartjs id="companyChart" >}}
new Chart(document.getElementById('companyChart'), {
    type: 'bar',
    data: {
        labels: ['EPAM', 'Capgemini', 'DaCodes', 'Dev.Pro', 'Workana', 'Maxana', 'Sezzle', 'Pavago', 'Infosys', 'Jalasoft', 'Multiplica Talent', 'Activate Talent', 'CI&T', 'Amazon Web Services'],
        datasets: [{
            data: [155, 75, 63, 50, 43, 40, 39, 37, 34, 33, 32, 30, 28, 27],
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

**9.7% of listings (487 jobs) come from staffing agencies.** The country-level breakdown: **Argentina (18.4%), Peru (16.5%), and Chile (13.2%)** have the heaviest agency presence; Brazil (7.0%) and Mexico (7.5%) are much cleaner markets.

{{< chartjs id="agencyCountryChart" >}}
new Chart(document.getElementById('agencyCountryChart'), {
    type: 'bar',
    data: {
        labels: ['Argentina', 'Peru', 'Chile', 'Costa Rica', 'Colombia', 'Mexico', 'Brazil'],
        datasets: [{
            label: 'Agency %',
            data: [18.4, 16.5, 13.2, 12.7, 12.2, 7.5, 7.0],
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

### Agencies also have worse Glassdoor ratings

Agencies are a major presence in the LATAM Cloud/DevOps market, but they don't have a great reputation. Among listings with Glassdoor ratings (1,803 direct employers vs 90 staffing firms), agencies score worse in every dimension, with the biggest gaps in **compensation (3.42 vs 2.92, -0.50)** and **culture (3.62 vs 3.08, -0.54)**.

{{< chartjs id="agencyRatingsChart" >}}
new Chart(document.getElementById('agencyRatingsChart'), {
    type: 'bar',
    data: {
        labels: ['Overall', 'Work/Life', 'Compensation', 'Culture', 'Career Opps'],
        datasets: [
            { label: 'Direct Employer', data: [3.72, 3.64, 3.42, 3.62, 3.45], backgroundColor: '#64748b', borderRadius: 3 },
            { label: 'Staffing Agency',  data: [3.55, 3.22, 2.92, 3.08, 2.99], backgroundColor: '#fb7185', borderRadius: 3 },
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

- **Learn Terraform.** It's the dominant IaC tool (1,579 mentions, 3x anything else), and Terraform Associate is the #2 most-requested certification, behind only AWS SA Associate.
- **Don't sleep on Azure.** Azure is closer to AWS than most people think (2,235 vs 2,576 total; Azure is actually ahead in Mexico, Peru, Costa Rica, and in junior-level roles across LATAM). In enterprise-heavy markets, Azure is the pragmatic first cloud to learn.
- **Python first, everything else second.** Python appears in 2,034 listings, 2.5x JS/TS, 3.7x Go, 54x Rust.
- **Kubernetes is expected.** 37% of roles overall mention K8s, rising to 62% at staff level. Get comfortable with it early.
- **Watch out for agencies.** 9.7% of listings are from staffing agencies, and they rate meaningfully worse on Glassdoor across *every* dimension, compensation especially (2.92 vs 3.42, a -15% gap). On Workable, roughly one in three listings is an agency. Always verify you're talking to the actual employer.
- **Junior roles are scarce.** Only 2.1% of listings are entry-level, and most ask for 1–2 years of experience anyway. Certifications (AWS SA Associate, Terraform Associate, Azure Administrator Associate / AZ-104) are one of the few signals that will move you past resume filters.
- **English matters.** 77.5% of listings in this dataset are written in English. Even roles based in Brazil or Mexico often publish in English first. Being comfortable communicating in technical English roughly triples your addressable pool compared to Spanish or Portuguese only.
- **Remote is the plurality, not the default.** 56% remote, 24% onsite, 21% hybrid. If you want remote, Costa Rica, Argentina, and Colombia are your best markets; Brazil, Chile, and Mexico still skew onsite/hybrid.
