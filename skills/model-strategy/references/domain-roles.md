# Domain Role Templates

This reference provides role templates organized by business domain. When generating a staffing
plan, select the relevant domains and adapt these templates to the user's specific scope.

Not every role is needed for every company. Use the user's scope answers to pick the right subset.
Roles can span multiple domains — a "Data Analyst" might serve both Finance and Marketing.

---

## Software Engineering & Delivery

| Role ID | Title | Description | Key Requirements |
|---|---|---|---|
| `principal_engineer` | Principal Engineer / Architect | System design decisions, architecture reviews, complex trade-offs | Highest intelligence, strong reasoning |
| `senior_fullstack_dev` | Senior Full Stack Developer | Feature implementation, API development, database work | Strong code generation, good SWE-bench |
| `senior_frontend_dev` | Senior Front End Developer | UI components, responsive layouts, design system implementation | Sustained iteration, component architecture |
| `senior_backend_dev` | Senior Backend Developer | Services, data pipelines, system integrations | Strong code gen, long context for large codebases |
| `ux_designer` | UX Designer | User flow design, UX audit, accessibility review, design specs | Multimodal (screenshot analysis), strong reasoning |
| `cloud_engineer` | Cloud Engineer / DevOps | Infrastructure-as-code, CI/CD, deployment, cost optimization | Long context for full IaC stacks, tool calling |
| `qa_engineer` | QA Engineer / Code Reviewer | Test generation, code review, coverage analysis | Cheap volume, escalation for security |
| `security_engineer` | Security Engineer | Security reviews, threat modeling, vulnerability analysis | Deep reasoning, long context |
| `mobile_dev` | Mobile Developer | iOS/Android/cross-platform app development | Code gen, platform-specific knowledge |
| `ml_engineer` | ML / AI Engineer | Model training, data pipelines, MLOps | Math/stats strength, code gen |
| `data_engineer` | Data Engineer | ETL pipelines, data warehousing, query optimization | SQL strength, infrastructure knowledge |
| `tech_writer` | Technical Writer | API docs, runbooks, onboarding guides, READMEs | Clear instruction following, structured output |

### Scope Variants
- **Full-stack web**: principal_engineer + senior_fullstack_dev + ux_designer + qa_engineer + cloud_engineer
- **API / backend only**: principal_engineer + senior_backend_dev + qa_engineer + cloud_engineer
- **Frontend only**: senior_frontend_dev + ux_designer + qa_engineer
- **Infrastructure**: principal_engineer + cloud_engineer + security_engineer
- **ML/AI**: ml_engineer + data_engineer + principal_engineer + cloud_engineer

---

## Finance & Accounting

| Role ID | Title | Description | Key Requirements |
|---|---|---|---|
| `chief_accountant` | Chief Accountant | Month-end close, financial statements, GAAP compliance | Strong reasoning, precision, structured output |
| `senior_accountant` | Senior Accountant | Journal entries, reconciliations, accruals | Reliable instruction following, math strength |
| `financial_analyst` | Financial Analyst / FP&A | Variance analysis, forecasting, budget vs. actual | Analytical reasoning, data processing |
| `tax_specialist` | Tax Specialist | Tax compliance, planning, return preparation | Domain knowledge, regulatory awareness |
| `auditor` | Internal Auditor | SOX testing, control assessments, sample selection | Methodical, structured output |
| `treasury_analyst` | Treasury Analyst | Cash flow forecasting, banking operations | Math strength, financial modeling |
| `bookkeeper` | Bookkeeper | Transaction categorization, data entry, bank feeds | Cheap volume, reliable accuracy |
| `payroll_specialist` | Payroll Specialist | Payroll processing, benefits administration | Precision, compliance awareness |

### Scope Variants
- **Bookkeeping**: bookkeeper + senior_accountant
- **Full accounting**: chief_accountant + senior_accountant + bookkeeper + payroll_specialist
- **FP&A**: financial_analyst + chief_accountant
- **Audit**: auditor + chief_accountant
- **Full finance dept**: all roles

---

## Legal & Compliance

| Role ID | Title | Description | Key Requirements |
|---|---|---|---|
| `general_counsel` | General Counsel | Legal strategy, risk assessment, complex negotiations | Highest intelligence, deep reasoning |
| `contract_reviewer` | Contract Reviewer | Clause analysis, redlining, playbook comparison | Long context, structured output |
| `compliance_officer` | Compliance Officer | Regulatory checks, policy review, jurisdiction analysis | Domain expertise, thoroughness |
| `ip_specialist` | IP Specialist | Patent review, trademark analysis, IP strategy | Specialized knowledge, reasoning |
| `paralegal` | Paralegal | Document preparation, research, filing | Reliable volume work, instruction following |
| `privacy_officer` | Privacy / DPO | GDPR, CCPA, data protection impact assessments | Regulatory knowledge, structured analysis |
| `litigation_support` | Litigation Support | Case research, document review, deposition prep | Long context, analytical reasoning |

### Scope Variants
- **Contract review**: contract_reviewer + paralegal
- **Compliance**: compliance_officer + privacy_officer
- **Full legal dept**: general_counsel + contract_reviewer + compliance_officer + paralegal

---

## Marketing & Content

| Role ID | Title | Description | Key Requirements |
|---|---|---|---|
| `marketing_strategist` | Marketing Strategist | Campaign planning, competitive analysis, positioning | Strong reasoning, creative thinking |
| `content_writer` | Content Writer | Blog posts, social media, email newsletters, landing pages | Creative, good instruction following |
| `seo_specialist` | SEO Specialist | Keyword research, on-page optimization, content gaps | Analytical, web knowledge |
| `email_marketer` | Email Marketing Specialist | Drip campaigns, sequence design, A/B testing | Structured output, conversion awareness |
| `social_media_manager` | Social Media Manager | Platform-specific content, engagement, scheduling | Creative, platform knowledge |
| `brand_manager` | Brand Manager | Voice consistency, style guide enforcement, brand audits | Attention to detail, pattern matching |
| `copywriter` | Copywriter | Ad copy, CTAs, headlines, product descriptions | Creative, concise, persuasive |
| `performance_analyst` | Marketing Performance Analyst | Campaign metrics, reporting, optimization recommendations | Data analysis, structured output |

### Scope Variants
- **Content only**: content_writer + seo_specialist + copywriter
- **Full marketing**: marketing_strategist + content_writer + seo_specialist + email_marketer + performance_analyst
- **Brand-focused**: brand_manager + content_writer + copywriter

---

## Customer Support

| Role ID | Title | Description | Key Requirements |
|---|---|---|---|
| `support_lead` | Support Lead | Escalation handling, quality review, process improvement | Strong reasoning, empathy |
| `support_agent` | Support Agent | Ticket responses, troubleshooting, customer communication | Fast inference, good instruction following |
| `kb_author` | Knowledge Base Author | Article drafting, FAQ maintenance, documentation | Clear writing, structured output |
| `triage_specialist` | Triage Specialist | Ticket categorization, priority assignment, routing | Fast, cheap, pattern matching |
| `escalation_manager` | Escalation Manager | Complex issue resolution, cross-team coordination | Deep reasoning, context retention |

### Scope Variants
- **Basic support**: support_agent + triage_specialist
- **Full support team**: all roles

---

## E-Commerce & Retail

| Role ID | Title | Description | Key Requirements |
|---|---|---|---|
| `ecommerce_strategist` | E-Commerce Strategist | Pricing, merchandising, conversion optimization | Analytical, business reasoning |
| `product_catalog_manager` | Product Catalog Manager | Listings, descriptions, categorization, SEO | Volume, consistency, instruction following |
| `inventory_analyst` | Inventory Analyst | Stock optimization, demand forecasting, reorder points | Math, data analysis |
| `customer_experience` | Customer Experience Manager | Journey mapping, feedback analysis, retention | Empathy, analytical reasoning |
| `marketplace_specialist` | Marketplace Specialist | Multi-channel listing, platform compliance | Platform knowledge, structured output |

---

## Healthcare / Medical

| Role ID | Title | Description | Key Requirements |
|---|---|---|---|
| `clinical_analyst` | Clinical Analyst | Medical record review, clinical data analysis | Domain expertise, precision |
| `compliance_reviewer` | Healthcare Compliance Reviewer | HIPAA, regulatory compliance, audit preparation | Regulatory knowledge, thoroughness |
| `medical_writer` | Medical Writer | Clinical documentation, patient materials, research summaries | Domain expertise, clear writing |
| `billing_specialist` | Medical Billing Specialist | Coding review, claims analysis, denial management | Precision, regulatory knowledge |

---

## Education & Training

| Role ID | Title | Description | Key Requirements |
|---|---|---|---|
| `curriculum_designer` | Curriculum Designer | Course structure, learning objectives, assessment design | Pedagogical reasoning, structured output |
| `content_developer` | Educational Content Developer | Lesson plans, study materials, exercises | Creative, clear explanation |
| `assessment_specialist` | Assessment Specialist | Quiz/exam creation, rubric design, grading criteria | Precision, fairness |
| `tutor` | AI Tutor | Student Q&A, concept explanation, practice problems | Patient, adaptive, math strength |

---

## Data Science & Analytics

| Role ID | Title | Description | Key Requirements |
|---|---|---|---|
| `lead_data_scientist` | Lead Data Scientist | Methodology selection, model design, statistical rigor | Highest intelligence, math/stats |
| `data_analyst` | Data Analyst | SQL queries, dashboards, ad-hoc analysis | SQL strength, visualization |
| `bi_developer` | BI Developer | Dashboard development, ETL, reporting infrastructure | SQL, tool calling, structured output |
| `statistician` | Statistician | Hypothesis testing, experimental design, causal inference | Math strength, methodological rigor |

---

## Sales & CRM

| Role ID | Title | Description | Key Requirements |
|---|---|---|---|
| `sales_strategist` | Sales Strategist | Pipeline analysis, territory planning, deal strategy | Business reasoning, data analysis |
| `sales_enablement` | Sales Enablement Specialist | Battlecards, pitch decks, competitive intel | Research, structured output |
| `crm_analyst` | CRM Analyst | Pipeline reporting, forecasting, data hygiene | Data analysis, SQL |
| `proposal_writer` | Proposal / RFP Writer | Proposal drafting, RFP responses, case studies | Clear writing, persuasion |

---

## HR & Recruiting

| Role ID | Title | Description | Key Requirements |
|---|---|---|---|
| `recruiter` | Recruiter | Job descriptions, candidate screening, outreach | Empathy, pattern matching |
| `hr_analyst` | HR Analyst | Compensation analysis, workforce planning | Data analysis, fairness |
| `policy_writer` | HR Policy Writer | Employee handbook, policy drafting, compliance | Regulatory knowledge, clear writing |
| `onboarding_specialist` | Onboarding Specialist | New hire guides, training materials, checklists | Structured output, thoroughness |

---

## Operations & Supply Chain

| Role ID | Title | Description | Key Requirements |
|---|---|---|---|
| `operations_manager` | Operations Manager | Process optimization, workflow design, efficiency analysis | Analytical, systems thinking |
| `supply_chain_analyst` | Supply Chain Analyst | Demand forecasting, vendor management, logistics | Math, data analysis |
| `procurement_specialist` | Procurement Specialist | Vendor evaluation, contract negotiation, cost analysis | Analytical, structured output |
| `quality_manager` | Quality Manager | QA processes, defect analysis, continuous improvement | Methodical, data-driven |

---

## Executive / Strategy

| Role ID | Title | Description | Key Requirements |
|---|---|---|---|
| `chief_of_staff` | Chief of Staff | Cross-functional coordination, strategic planning, OKR tracking | Highest intelligence, broad knowledge |
| `strategy_analyst` | Strategy Analyst | Market research, competitive analysis, business cases | Research, analytical reasoning |
| `executive_assistant` | Executive Assistant | Meeting prep, email drafting, scheduling, briefings | Fast, reliable, instruction following |
| `communications_lead` | Communications Lead | Internal comms, status reports, stakeholder updates | Clear writing, structured output |

---

## Creative / Design

| Role ID | Title | Description | Key Requirements |
|---|---|---|---|
| `creative_director` | Creative Director | Visual strategy, brand direction, design review | Multimodal, strong reasoning |
| `graphic_designer` | Graphic Designer | Visual assets, layouts, presentations | Multimodal, creative |
| `ux_researcher` | UX Researcher | User research, surveys, usability testing, synthesis | Analytical, empathy |
| `content_designer` | Content Designer | Information architecture, microcopy, user flows | UX writing, structured thinking |

---

## Cross-Domain Utility Roles

These roles appear in almost every configuration regardless of domain:

| Role ID | Title | Description |
|---|---|---|
| `general_researcher` | General Researcher | Web research, fact-checking, competitive intelligence |
| `document_specialist` | Document Specialist | Report formatting, document conversion, template creation |
| `data_entry` | Data Entry / Processing | High-volume structured data tasks, form filling |
| `translator` | Translator / Localizer | Multi-language content, localization review |
| `meeting_assistant` | Meeting Assistant | Agenda prep, notes, action items, follow-ups |
