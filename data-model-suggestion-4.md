# Data Model Suggestion 4: Graph Database Model (Neo4j)

## Approach

A property graph model in Neo4j where experts, organisations, skills, industries, projects, and engagements are nodes connected by typed, weighted relationships. This approach models the expert network as a literal graph -- the most natural representation for a platform whose core value proposition is connecting people with knowledge seekers through multi-dimensional relationships.

## Why a Graph Database Suits an Expert Network Platform

The name says it all: an expert **network** platform is fundamentally a graph problem. The core operations are:

1. **Expert matching**: "Find experts who have skill X, worked in industry Y, speak language Z, are not on any restricted list for this client, and were highly rated by similar clients." This is a multi-hop graph traversal, not a JOIN.
2. **Relationship discovery**: "Which experts have we already engaged who are connected (same company, same industry, same skill cluster) to experts we haven't reached?"
3. **Compliance path checking**: "Does this expert have any path (via companies, industries, relationships) to entities on our restricted list?" Graph databases answer reachability questions in constant time per hop.
4. **Knowledge graphs**: The platform's AI features (semantic matching, knowledge gap analysis, cross-call synthesis) naturally operate on a knowledge graph of experts, topics, insights, and research briefs.
5. **Recommendation engine**: "Experts similar to those who performed well on this project" is a collaborative-filtering traversal.

Graph databases excel at these relationship-centric queries where relational databases would require complex multi-table JOINs with unpredictable depth. Neo4j specifically provides:

- **Cypher query language** optimized for pattern matching across relationships.
- **Native graph storage** with index-free adjacency for O(1) relationship traversal.
- **Graph Data Science library** with community detection, similarity algorithms, and path-finding built in.
- **Full-text and vector search** capabilities for hybrid semantic + graph matching.

---

## Node Definitions

```cypher
// ============================================================
// USER NODES
// ============================================================

// Base User node (all user types)
CREATE CONSTRAINT user_id_unique FOR (u:User) REQUIRE u.id IS UNIQUE;
CREATE CONSTRAINT user_email_unique FOR (u:User) REQUIRE u.email IS UNIQUE;

// User node properties:
// (:User {
//   id: UUID,
//   email: String,
//   firstName: String,
//   lastName: String,
//   passwordHash: String,
//   status: "pending"|"active"|"suspended"|"deactivated",
//   emailVerified: Boolean,
//   lastLoginAt: DateTime,
//   createdAt: DateTime,
//   updatedAt: DateTime
// })

// Users have additional labels for their type: :Expert, :Client, :Admin


// ============================================================
// EXPERT NODES (multi-label: User + Expert)
// ============================================================

CREATE INDEX expert_available FOR (e:Expert) ON (e.isAvailable);
CREATE INDEX expert_rating FOR (e:Expert) ON (e.avgRating);
CREATE INDEX expert_rate FOR (e:Expert) ON (e.hourlyRateCents);

// (:User:Expert {
//   id: UUID,
//   headline: String,
//   bio: String,
//   hourlyRateCents: Integer,
//   currency: String,  // "USD"
//   timezone: String,
//   isAvailable: Boolean,
//   avgRating: Float,
//   totalCalls: Integer,
//   totalEarningsCents: Integer,
//   verificationStatus: "unverified"|"pending"|"verified"|"rejected",
//   languages: [String],  // ["en", "fr"]
//   onboardedAt: DateTime,
//   // Embedding stored as list of floats for vector search
//   embedding: [Float]
// })


// ============================================================
// ORGANISATION NODES
// ============================================================

CREATE CONSTRAINT org_id_unique FOR (o:Organisation) REQUIRE o.id IS UNIQUE;
CREATE CONSTRAINT org_slug_unique FOR (o:Organisation) REQUIRE o.slug IS UNIQUE;

// (:Organisation {
//   id: UUID,
//   name: String,
//   slug: String,
//   orgType: "consulting"|"investment_fund"|"corporate"|"research"|"other",
//   subscriptionTier: "payg"|"starter"|"professional"|"enterprise",
//   billingEmail: String,
//   status: String,
//   createdAt: DateTime
// })


// ============================================================
// SKILL NODES (taxonomy)
// ============================================================

CREATE CONSTRAINT skill_id_unique FOR (s:Skill) REQUIRE s.id IS UNIQUE;
CREATE CONSTRAINT skill_slug_unique FOR (s:Skill) REQUIRE s.slug IS UNIQUE;
CREATE TEXT INDEX skill_name_text FOR (s:Skill) ON (s.name);

// (:Skill {
//   id: UUID,
//   name: String,
//   slug: String,
//   description: String,
//   createdAt: DateTime
// })

// (:SkillCategory {
//   id: UUID,
//   name: String,
//   description: String,
//   sortOrder: Integer
// })


// ============================================================
// INDUSTRY NODES
// ============================================================

CREATE CONSTRAINT industry_id_unique FOR (i:Industry) REQUIRE i.id IS UNIQUE;

// (:Industry {
//   id: UUID,
//   name: String,
//   sicCode: String,
//   naicsCode: String
// })


// ============================================================
// COMPANY NODES (work history references)
// ============================================================

CREATE CONSTRAINT company_id_unique FOR (c:Company) REQUIRE c.id IS UNIQUE;
CREATE TEXT INDEX company_name_text FOR (c:Company) ON (c.name);

// (:Company {
//   id: UUID,
//   name: String,
//   domain: String,
//   industry: String
// })


// ============================================================
// PROJECT NODES
// ============================================================

CREATE CONSTRAINT project_id_unique FOR (p:Project) REQUIRE p.id IS UNIQUE;

// (:Project {
//   id: UUID,
//   title: String,
//   description: String,
//   brief: String,
//   status: "draft"|"active"|"paused"|"completed"|"archived",
//   budgetCents: Integer,
//   currency: String,
//   startDate: Date,
//   endDate: Date,
//   briefEmbedding: [Float],
//   createdAt: DateTime
// })


// ============================================================
// ENGAGEMENT NODES
// ============================================================

CREATE CONSTRAINT engagement_id_unique FOR (e:Engagement) REQUIRE e.id IS UNIQUE;

// (:Engagement {
//   id: UUID,
//   status: "requested"|"expert_invited"|"accepted"|"compliance_pending"|
//           "compliance_cleared"|"scheduled"|"completed"|"declined"|"cancelled",
//   matchScore: Float,
//   hourlyRateCents: Integer,
//   currency: String,
//   notes: String,
//   createdAt: DateTime
// })


// ============================================================
// CALL NODES
// ============================================================

CREATE CONSTRAINT call_id_unique FOR (cl:Call) REQUIRE cl.id IS UNIQUE;

// (:Call {
//   id: UUID,
//   scheduledStart: DateTime,
//   scheduledEnd: DateTime,
//   actualStart: DateTime,
//   actualEnd: DateTime,
//   durationMinutes: Integer,
//   status: "scheduled"|"in_progress"|"completed"|"cancelled"|"no_show",
//   meetingProvider: String,
//   meetingUrl: String,
//   recordingUrl: String,
//   createdAt: DateTime
// })


// ============================================================
// TRANSCRIPT NODES
// ============================================================

CREATE CONSTRAINT transcript_id_unique FOR (t:Transcript) REQUIRE t.id IS UNIQUE;
CREATE FULLTEXT INDEX transcript_fulltext FOR (t:Transcript) ON EACH [t.rawText];

// (:Transcript {
//   id: UUID,
//   rawText: String,
//   language: String,
//   wordCount: Integer,
//   confidenceScore: Float,
//   createdAt: DateTime
// })

// (:Summary {
//   id: UUID,
//   summaryType: "executive"|"expert_background"|"sentiment"|"key_takeaways"|"thematic",
//   content: String,
//   modelVersion: String,
//   createdAt: DateTime
// })

// (:Topic {
//   id: UUID,
//   name: String,
//   description: String
// })


// ============================================================
// COMPLIANCE NODES
// ============================================================

CREATE CONSTRAINT compliance_check_id FOR (cc:ComplianceCheck) REQUIRE cc.id IS UNIQUE;

// (:ComplianceCheck {
//   id: UUID,
//   checkType: "nda"|"conflict_of_interest"|"mnpi_acknowledgement"|"restricted_list"|"gdpr_consent",
//   status: "pending"|"passed"|"failed"|"waived"|"expired",
//   automated: Boolean,
//   details: String,
//   checkedAt: DateTime,
//   expiresAt: DateTime
// })

// (:RestrictedEntity {
//   id: UUID,
//   entityType: "company"|"person"|"ticker"|"topic",
//   entityName: String,
//   reason: String,
//   validFrom: Date,
//   validUntil: Date
// })

// (:NDADocument {
//   id: UUID,
//   documentUrl: String,
//   signedByExpert: Boolean,
//   signedByClient: Boolean,
//   expertSignedAt: DateTime,
//   clientSignedAt: DateTime
// })


// ============================================================
// SURVEY NODES
// ============================================================

CREATE CONSTRAINT survey_id_unique FOR (s:Survey) REQUIRE s.id IS UNIQUE;

// (:Survey {
//   id: UUID,
//   title: String,
//   status: "draft"|"active"|"closed"|"archived",
//   opensAt: DateTime,
//   closesAt: DateTime
// })

// (:Question {
//   id: UUID,
//   text: String,
//   questionType: "text"|"single_choice"|"multi_choice"|"rating"|"ranking",
//   sortOrder: Integer,
//   isRequired: Boolean,
//   options: [String]  // for choice questions
// })

// (:Response {
//   id: UUID,
//   submittedAt: DateTime
// })


// ============================================================
// PAYMENT NODES
// ============================================================

CREATE CONSTRAINT invoice_id_unique FOR (inv:Invoice) REQUIRE inv.id IS UNIQUE;

// (:Invoice {
//   id: UUID,
//   invoiceNumber: String,
//   status: "draft"|"issued"|"paid"|"overdue"|"cancelled"|"refunded",
//   subtotalCents: Integer,
//   taxCents: Integer,
//   totalCents: Integer,
//   currency: String,
//   issuedAt: DateTime,
//   dueAt: DateTime,
//   paidAt: DateTime
// })

// (:Payout {
//   id: UUID,
//   amountCents: Integer,
//   currency: String,
//   status: "pending"|"processing"|"paid"|"failed",
//   paymentMethod: String,
//   paidAt: DateTime
// })
```

---

## Relationship Definitions

```cypher
// ============================================================
// CORE RELATIONSHIPS
// ============================================================

// Organisation membership
// (user:User)-[:MEMBER_OF {role: "owner"|"admin"|"manager"|"member", since: DateTime}]->(org:Organisation)

// Expert skills (weighted, proficiency-annotated)
// (expert:Expert)-[:HAS_SKILL {proficiency: "expert", years: 8, endorsements: 12}]->(skill:Skill)

// Skill taxonomy hierarchy
// (skill:Skill)-[:BELONGS_TO]->(category:SkillCategory)
// (child:SkillCategory)-[:CHILD_OF]->(parent:SkillCategory)
// (skill1:Skill)-[:RELATED_TO {strength: 0.85}]->(skill2:Skill)

// Expert industry coverage
// (expert:Expert)-[:COVERS_INDUSTRY {isPrimary: true, years: 15}]->(industry:Industry)
// (child:Industry)-[:SUB_INDUSTRY_OF]->(parent:Industry)

// Expert work history
// (expert:Expert)-[:WORKED_AT {title: "VP Engineering", startDate: date("2020-01"), endDate: date("2024-06"), isCurrent: false}]->(company:Company)
// (company:Company)-[:IN_INDUSTRY]->(industry:Industry)

// Project ownership and requirements
// (org:Organisation)-[:OWNS_PROJECT]->(project:Project)
// (user:User)-[:CREATED]->(project:Project)
// (project:Project)-[:REQUIRES_SKILL {importance: "required"}]->(skill:Skill)
// (project:Project)-[:TARGETS_INDUSTRY]->(industry:Industry)

// Engagements connect projects to experts
// (project:Project)-[:HAS_ENGAGEMENT]->(engagement:Engagement)
// (engagement:Engagement)-[:WITH_EXPERT]->(expert:Expert)
// (user:User)-[:REQUESTED]->(engagement:Engagement)

// AI matching (stored as relationship with score)
// (project:Project)-[:MATCHED_TO {score: 0.9423, matchedSkills: ["uuid"], modelVersion: "v3", matchedAt: DateTime}]->(expert:Expert)

// Compliance
// (engagement:Engagement)-[:HAS_CHECK]->(check:ComplianceCheck)
// (check:ComplianceCheck)-[:PERFORMED_BY]->(user:User)
// (org:Organisation)-[:HAS_RESTRICTED]->(entity:RestrictedEntity)
// (entity:RestrictedEntity)-[:ADDED_BY]->(user:User)
// (engagement:Engagement)-[:HAS_NDA]->(nda:NDADocument)

// Calls and transcripts
// (engagement:Engagement)-[:HAS_CALL]->(call:Call)
// (call:Call)-[:HAS_TRANSCRIPT]->(transcript:Transcript)
// (transcript:Transcript)-[:HAS_SUMMARY]->(summary:Summary)
// (transcript:Transcript)-[:DISCUSSES]->(topic:Topic)
// (summary:Summary)-[:MENTIONS]->(topic:Topic)

// Cross-call synthesis
// (project:Project)-[:HAS_SYNTHESIS]->(synthesis:SynthesisReport)
// (synthesis:SynthesisReport)-[:SYNTHESIZES]->(call:Call)

// Surveys
// (project:Project)-[:HAS_SURVEY]->(survey:Survey)
// (survey:Survey)-[:HAS_QUESTION]->(question:Question)
// (expert:Expert)-[:RESPONDED_TO {submittedAt: DateTime}]->(survey:Survey)

// Ratings (as relationships, not nodes)
// (user:User)-[:RATED {rating: 5, expertise: 5, communication: 4, feedback: "...", createdAt: DateTime}]->(call:Call)

// Payments
// (org:Organisation)-[:BILLED_VIA]->(invoice:Invoice)
// (invoice:Invoice)-[:COVERS_CALL]->(call:Call)
// (expert:Expert)-[:RECEIVES]->(payout:Payout)
// (payout:Payout)-[:FOR_CALL]->(call:Call)
```

---

## Key Cypher Queries

```cypher
// ============================================================
// 1. SEMANTIC + GRAPH EXPERT MATCHING
// Find experts matching a project brief, factoring in skills,
// industry, availability, compliance, and past ratings
// ============================================================

MATCH (p:Project {id: $projectId})-[:REQUIRES_SKILL]->(reqSkill:Skill)
MATCH (expert:Expert)-[hs:HAS_SKILL]->(reqSkill)
WHERE expert.isAvailable = true
  AND expert.verificationStatus = 'verified'
  AND hs.proficiency IN ['advanced', 'expert']
WITH expert, p, count(reqSkill) AS matchedSkills,
     avg(hs.years) AS avgSkillYears
// Check expert is not restricted
OPTIONAL MATCH (p)<-[:OWNS_PROJECT]-(org:Organisation)-[:HAS_RESTRICTED]->(restricted:RestrictedEntity)
WHERE restricted.entityName IN [
  [(expert)-[:WORKED_AT]->(c:Company) | c.name]
]
WITH expert, matchedSkills, avgSkillYears, count(restricted) AS restrictionHits
WHERE restrictionHits = 0
// Factor in past ratings
OPTIONAL MATCH (rater:User)-[r:RATED]->(call:Call)<-[:HAS_CALL]-(:Engagement)-[:WITH_EXPERT]->(expert)
WITH expert, matchedSkills, avgSkillYears, avg(r.rating) AS avgRating
RETURN expert.id, expert.headline, expert.hourlyRateCents,
       matchedSkills, avgSkillYears, coalesce(avgRating, 0) AS avgRating
ORDER BY matchedSkills DESC, avgRating DESC, avgSkillYears DESC
LIMIT 20;


// ============================================================
// 2. COMPLIANCE PATH CHECK
// Check if an expert has any connection to restricted entities
// ============================================================

MATCH (org:Organisation {id: $orgId})-[:HAS_RESTRICTED]->(restricted:RestrictedEntity)
MATCH (expert:Expert {id: $expertId})
OPTIONAL MATCH path = (expert)-[:WORKED_AT|COVERS_INDUSTRY*1..3]-(restricted)
RETURN restricted.entityName, restricted.entityType, restricted.reason,
       CASE WHEN path IS NOT NULL THEN true ELSE false END AS connected,
       [node IN nodes(path) | labels(node)[0] + ': ' + coalesce(node.name, node.entityName, '')] AS connectionPath;


// ============================================================
// 3. KNOWLEDGE GAP ANALYSIS
// Find skills required by a project that no engaged expert covers
// ============================================================

MATCH (p:Project {id: $projectId})-[:REQUIRES_SKILL]->(reqSkill:Skill)
OPTIONAL MATCH (p)-[:HAS_ENGAGEMENT]->(eng:Engagement {status: 'completed'})-[:WITH_EXPERT]->(expert:Expert)-[:HAS_SKILL]->(reqSkill)
WITH reqSkill, count(expert) AS expertsCovering
WHERE expertsCovering = 0
RETURN reqSkill.name AS uncoveredSkill, reqSkill.id
ORDER BY reqSkill.name;


// ============================================================
// 4. EXPERT SIMILARITY / RECOMMENDATION
// Find experts similar to highly-rated ones on a project
// ============================================================

MATCH (p:Project {id: $projectId})-[:HAS_ENGAGEMENT]->(eng:Engagement {status: 'completed'})-[:WITH_EXPERT]->(topExpert:Expert)
MATCH (rater)-[r:RATED]->(call:Call)<-[:HAS_CALL]-(eng)
WHERE r.rating >= 4
WITH topExpert, collect(DISTINCT topExpert) AS topExperts
MATCH (topExpert)-[:HAS_SKILL]->(sharedSkill:Skill)<-[:HAS_SKILL]-(similar:Expert)
WHERE NOT similar IN topExperts
  AND similar.isAvailable = true
  AND similar.verificationStatus = 'verified'
WITH similar, count(DISTINCT sharedSkill) AS sharedSkills,
     collect(DISTINCT sharedSkill.name) AS skillNames
ORDER BY sharedSkills DESC
LIMIT 10
RETURN similar.id, similar.headline, similar.hourlyRateCents,
       similar.avgRating, sharedSkills, skillNames;


// ============================================================
// 5. TOPIC KNOWLEDGE GRAPH
// Build a topic map from all transcripts in a project
// ============================================================

MATCH (p:Project {id: $projectId})-[:HAS_ENGAGEMENT]->(eng)-[:HAS_CALL]->(call:Call)
      -[:HAS_TRANSCRIPT]->(t:Transcript)-[:DISCUSSES]->(topic:Topic)
WITH topic, count(DISTINCT t) AS mentionCount,
     collect(DISTINCT call.id) AS callIds
ORDER BY mentionCount DESC
RETURN topic.name, mentionCount, callIds
LIMIT 50;


// ============================================================
// 6. ENGAGEMENT LIFECYCLE TIMELINE
// Full audit trail for a specific engagement
// ============================================================

MATCH (eng:Engagement {id: $engagementId})
OPTIONAL MATCH (eng)-[:WITH_EXPERT]->(expert:Expert)
OPTIONAL MATCH (eng)-[:HAS_CHECK]->(check:ComplianceCheck)
OPTIONAL MATCH (eng)-[:HAS_CALL]->(call:Call)
OPTIONAL MATCH (call)-[:HAS_TRANSCRIPT]->(t:Transcript)-[:HAS_SUMMARY]->(s:Summary)
OPTIONAL MATCH (eng)-[:HAS_NDA]->(nda:NDADocument)
RETURN eng, expert, collect(DISTINCT check) AS checks,
       collect(DISTINCT call) AS calls,
       collect(DISTINCT {transcript: t.id, summaryType: s.summaryType}) AS transcripts,
       nda;
```

---

## Complementary Storage (Polyglot Persistence)

A pure graph database is not ideal for every workload in this platform. The recommended architecture uses Neo4j as the primary graph alongside complementary stores:

| Workload | Store | Reason |
|----------|-------|--------|
| Expert graph, matching, compliance paths | Neo4j | Relationship traversal is the core operation |
| Full-text transcript search | Elasticsearch | Better full-text search at scale than Neo4j's built-in |
| Call recordings, documents | S3/Object storage | Binary large objects |
| Transactional payments, invoices | PostgreSQL | ACID guarantees for financial data |
| Session state, caching | Redis | Low-latency reads |
| Embeddings / vector search | Neo4j vector index or Weaviate | Semantic similarity matching |
| Audit log (append-only, high-volume) | PostgreSQL or ClickHouse | Time-series append workload |

Data synchronization between stores uses change data capture (CDC) from Neo4j's transaction log or application-level dual writes with outbox pattern.

---

## Trade-offs

**Strengths:**
- **Natural domain fit**: expert networks are literally graphs; the data model matches the mental model.
- **Multi-hop queries in constant time**: compliance path checks, similarity discovery, and knowledge gap analysis are first-class operations, not expensive JOINs.
- **Schema flexibility**: adding new node types, relationship types, or properties requires no migration.
- **Built-in graph algorithms**: Neo4j GDS provides PageRank (expert influence), community detection (expert clusters), similarity (collaborative filtering), and shortest path (compliance risk) out of the box.
- **Knowledge graph ready**: the topic/transcript/expert graph is a natural foundation for GraphRAG integration with LLMs.

**Weaknesses:**
- **No native ACID across stores**: the polyglot architecture requires careful consistency management between Neo4j, PostgreSQL, and Elasticsearch.
- **Aggregate queries are slow**: "total revenue by month" or "average call duration by organisation" are better served by a relational or columnar database.
- **Smaller ecosystem**: fewer ORM libraries, fewer developers experienced with Cypher, fewer managed hosting options than PostgreSQL.
- **Storage overhead**: property graphs store relationship metadata per edge, which is less space-efficient than foreign keys for simple 1:N relationships.
- **Operational complexity**: running and maintaining multiple data stores increases infrastructure burden.

## Scalability Considerations

- **Neo4j Aura (managed)**: handles clustering, replication, and backups; avoids operational overhead.
- **Read replicas**: Neo4j supports causal clustering with read replicas for scaling query workloads.
- **Sharding**: Neo4j 5+ supports composite databases for distributing graphs across servers.
- **Graph size**: Neo4j handles billions of nodes and relationships; an expert network platform with millions of experts is well within capacity.
- **Vector indexes**: Neo4j's built-in vector search supports semantic matching without a separate vector database for moderate-scale workloads.

## Migration Path

Start with Neo4j for the expert graph, matching, and compliance domains. Use PostgreSQL for payments and audit logging from day one. If the platform scales beyond Neo4j's full-text search capabilities, add Elasticsearch. If the team finds the polyglot architecture too complex, the graph data can be partially denormalized into PostgreSQL with JSONB columns (see Suggestion 3) while retaining Neo4j for the matching and compliance path-checking workloads where it provides the most value.
