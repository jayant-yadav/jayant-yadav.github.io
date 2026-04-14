---
layout: post
title: Custom Agent for UNICEF Documents
description: A RAG QA system for public UNICEF documents for Innovation specific responses. 
Date: 2026-03-10 15:01:35 +0300
image: '/images/posts/post-2/custom_rag.png'
image_caption: 'Custom UNICEF agent'
tags: [RAG, agent, project]
published: true
---

## Problem
Country Office Annual Reports (COAR) summarizes the efforts and outcomes of various programmes planned and in-affect in all the countries where UNICEF has a presence. These are produced by the Country Offices (COs), Regional Offices (ROs) and Headquater divisions (HQ) each year. Natrually, they contain weatlh of information on what strategies are working, which needs to be re-evaluated or improved. These reports contain extensive cross reference among programmes/countries, prognosis over the years and funding information for these programmes. The Offices are also reporting them in 3 languages viz Engligh, French and Spanish. 

Reading and understanding them to derive insights for innovation is a labourous task, which is only gonig to grow over the years. This calls for automation into condensing information and grounding them with factual instances mentioned in ~1000 pager reports that are generated each year. 

## Solution
With the advent of LLMs around 2023, it was possible to consturct information retrieval systems that can understand the context, condense and extract passages that are relavent for innovation-related analysis, beyond just keyword lookups. This is where Retrival Augemented Generation (RAG) powered by LLMs could be employed. 

This technique requries to chunk COAR reports into smaller passages that can be converted into embeddings, an AI readable string of numbers, that also carry contextual information. A user query can then be mapped with them to find the most "similar" passages addressing that query. The most relevant passages are then drawn out of the sea of chunks and stringed together with the help of an LLM, that now has more factual information to answer the user's query. This methodology of retrieval (of most relevant chunks) and generation (of a reply to a user's query) is termed RAG. 

There can be many modifications done to a RAG system but a simplest form would look like so:

![Vanilla RAG](/images/posts/post-2/vanilla_rag.png)

Building on this Vanilla RAG system, we can customize it to address our innovation-related queries such as :
* How did UNICEF build scalable models for innovation in xyz portfolio?
* What partnerships did UNICEF form for innovation in xyz setting? 
* What challenges did UNICEF face in building a scalable model in xyz portfolio?
* and many more...

To equip LLM with the understanding of what UNICEF means by innovation, and how to prepare a short report on each of the above user queries, custom prompts can be fed in, along with a summarization of innovations found across each chunk that were ealier extracted from the reports. This can be visualized in the following manner:

![Custom RAG](/images/posts/post-2/custom_rag.png)

The motivation to perform a summarization before LLM infers from the chunks is two folds:
1. LLM does not recognize what Innovation means for UNICEF in particular. It might apply its general knowledge instead of following UNICEF's core principals of Innovaiton. 
2. LLM needs to stay grounded to the facts that are mentioned in the chunks. When it sees the summary of the chunk based on the Innovation definition set by UNICEF, along with the original chunk, we believe it will be better equipped to answer queries that require it to reason out how a portfolio showcased innovation during its implementation and growth phase. 

The Custom RAG is also prompted differenently for different type of queries. Latest LLMs like GPT-5, are capable of thinking and reasoning. Giving specific prompts to plan out a reponse before answering a portfolio related query gives the LLM the ability to perform a checklist of operations and avoid hallucinating when answers are not present in the context provided. 

Lastly, a good routing mechanism is needed to route the user query to these appropriate LLM prompts. This needs a query classifier, which again with the help of an smaller LLM can be put into action. To avoid vague queries, a smaller LLM is also asked to elaborate on the query by adding keywords, so that the elaborated query is easier to match with relavent chunks. For example, quering about EMEA region vs quering about EMEA region with countries such as Bosnia, Lithuania, Croatia etc, gives the query retriever the ability to also look for country names rather than just "EMEA" as a keyword. Similarly, asking about scalability of a model could mean differently for Humanitarian setting vs Gender Equality setting. Giving a few more keywords by rewriting the query is helpful for the retriver model when "scalability" as a keyword is not directly mentioned in the excerpts. This rewriten query is also multi-lingualized in French and Spanish to hit chunks that are in different languages.

## Technical
Experimentation in two ways was conducted of building this “ask questions over reports” assistant. Both follow the same idea: (1) find the most relevant passages in the reports, then (2) ask an LLM to write a concise answer using only that evidence.

* Approach A — Custom RAG pipeline (LlamaIndex + Milvus):
    Building an end-to-end pipeline: clean the report text, split it into smaller chunks, convert each chunk into an embedding (a numeric “fingerprint” of meaning), and store those embeddings in a vector database (Milvus). At question time, the system retrieves the most similar chunks and the LLM writes an answer grounded in those excerpts. Pre-generated document-level summaries were also generated to answer “pattern / cross-country” questions faster.

* Approach B — RAGFlow conversational agent (final approach):
    Using RAGFlow platform to create a conversational agent that first understands the user’s question type (e.g., evolution over time, scalable models, challenges, partnerships), then rewrites the question to make it easier to search, retrieves a small set of relevant excerpts, and finally produces a structured answer. Similar to approach A, pre-generated document-level summaries were generated.

In the end, RAGFlow approach was used because it gave a faster path to a usable, maintainable agent with clear routing and consistent answer formats. Since its community maintained, it will be easy to deploy on UNICEF infrastructure and update in regular intervals.

RAGFlow instance for this work was hosted on AWS EC2 instance with following specifications:
* Instance: c5a.4xlarge, vCPUs: 16
* Volume: gp3, 50 GiB
* OpenAI embedding models and LLM used : text-embedding-3-large, GPT-4.1-nano, GPT-5-mini and GPT-5.2. 


### RAGFlow agent architecture

![EYSN agent RAGFlow](/images/posts/post-2/EYSN_agent.png)

Agent components:
* Dataset ingestion:
The ingestion pipeline consists of an OCR reader and the EYSN reports are provided in pdf formats. This is where the pipeline also generates a summarization based on the UNICEF's innovation definition, via a method called [RAPTOR](https://arxiv.org/html/2401.18059v1). OpenAI text-embedding-3-large embedding model is used to vectorize all the documents that are parsed into [Infinity](https://github.com/infiniflow/infinity) Vector Database.
* Query Optimization:
GPT-4.1-nano is used to rewrite the query and it is instructed to elaborate the query to make it easier to simlarity search in a database. This results in expanded arconyms and synonym addition to the original query.
* Retrieval: 
The re-writen query and the original documents along with their summaries are matched using similarity and keyword matching. A cross-lingual search is also performed by writing the query into English, French and Spanish to match with document chunks that are in written in these languages. Top 20 chunks of 200-300 words each are fetched in this process.
* Query categorization and routing:
To route the query to a prompt engineered LLM, the original user's query is also categorized into 6 different categories: topic evolution, scalable model, challenges, partnerships, miscellenous UNICEF related and casual chat. GPT-4.1-nano is deployed for this task again.
* Answer generation:
GPT-5.1-mini and GPT-5.2 are prompt engineered to answer user's original query by providing them with highly tuned instructions and the retrieved document chunks. This stage also combines 2 different answers by LLMs into one in a "waterfall" way, specifically for query related to building scalable models and its associated challenges. The answer is also coupled with citations so that the user can look at the chucks which were referred to answer that query.


### Challenges

Building a custom RAG agent came with a few challenges and the most notable one of them were:

* Fast evolving tooling:
    Techstack used for building a RAG system like this one requires us to be aware of the evolution of the softwares and techniques that support these systems. Most of these are community-led projects and therefore get nightly updates which might bring a new feature or completely break our existing tech pipeline. There were many instances when the our running code broke because of some update that was pushed over the weeked. At the same time, not updating these backend libararies is also not a good strategy as we miss the opportunity of using a good software update. Also, not updating them for a long time risks our softare to go out of date very quickly, given that the backend libraries are being updated at a very fast pace. A good balance of software update and freezing is required for this matter.

* Multi-linugual retrieval is harder than it sounds:
    On paper the logic seems to be straigtforward. Which is to generate multi-lingual versions of the query and perform a search in the database to find hits. The top matches can then be passed onto the LLM to generate a final answer. However, it was found that RAGFlow's cross-lingual search was not performing as it promised. After generating multi-lingual queries, it used to only return chunks for one of the languages all the time. Perhaps this will be addressed in the later versions of the application. For this reason, the answers genereted in the final report only comes from English chunks mostly. 

* LLM responses keeps changing:
    Reponse steering by proprietry LLMs like GPT-4 and 5 that were used in this work, are controlled by LLM providers only. This means that the kind of reponse we expect the LLM to give for a specific prompt might change over time. For this reason, the prompts need to be monitored and corrected throughout the lifecycle of the project. The biggest change in reponses are when a newer version of same family of LLMs are used, like going from GPT-4.3 to GPT-5.1. This brings a massive change in their reponses and their chain of thoughts. This is the price we pay for using latest LLMs with better thinking capaiblities and larger context windows.

* Infrastructure is expensive:
    Developing such a software never used to be so much memory hungry earlier, as the document parsing pipelines were very light. With LLMs inserted in between for data parsing and output generation, if building a completely in-house software to protect data leakage to third-party LLM providers such as OpenAI, the memory hungry LLMs need to be deployed on costly hardware (GPUs in this case). The costing for such a hardware limits our risk taking and prohibits us to run pipelines for pure exploration purposes only, where each hour is heavy on our total budget. 

## Results
The system was used to produce an *EYSN Insights Report* that synthesizes patterns in UNICEF innovation narratives across multiple years as a thematic scan, grounded in retrieved excerpts.

Full report: [EYSN_insights_report.md](./EYSN_insights_report.md)

Interesting findings from the report’s summary + conclusion:
* **Innovation is increasingly framed as system strengthening (not isolated pilots):** across sectors, “scale” shows up as institutionalization—embedding tools and operating models into routine government workflows (registries, case management/referral pathways, service packages, curricula and teacher development systems) so delivery survives shocks and staff/coordination changes.
* **Digital platforms are becoming the backbone for delivery and accountability:** repeated references to government-oriented information systems and shared platforms (e.g., DHIS2/EMIS-linked systems, PRIMERO/CPIMS+, helplines, open-source tools like CommCare, and education platforms like Learning Passport) suggest a shift toward standardizing workflows and shortening feedback loops—paired with the practical reality that buy-in, data governance, and monitoring capacity remain prerequisites.
* **Integration is operationalized via “connective tissue”:** cash/social protection delivery systems, coordination platforms (clusters/joint programmes/sector groups), and connectivity-for-learning initiatives show up as cross-cutting enablers that link households and communities to multi-service packages (“cash-plus”, referrals) and align multi-sector responses in both humanitarian and development contexts.
* **Partnerships and financing are consistently “braided” to make scale stick:** pooled/joint mechanisms, bilateral and multilateral support, IFI-linked financing, and private-sector partnerships recur as enabling conditions—while constrained fiscal space and the difficulty of defining sustainable “win–win” private-sector models are recurring bottlenecks.
* **Inclusion and prevention are increasingly built into scaled delivery systems:** innovation narratives more often embed disability inclusion, gender equality, youth/adolescents, and MHPSS into national platforms and minimum service packages, rather than treating them as stand-alone project components.

## Future Improvements

LLM landscape is evolving quickly with stronger models being released in the market. Some research shows that simantic matching in traditional RAG systems can be outperformed by coupling it with traditional search techniques like BM25 (something that Google used to find results for you pre-LLM era) and reranking models that ranks the retrieved chunks as per the user query. RAG agents can be made more sophesticated by allowing them to perform regex matching (think Ctrl+F in a PDF but more advanced) or executing code to parse through chunks, before they are simantically matched with the user query. This can give the agent time to iterate through the documents, extract based on keywords better, jump across documents based on their metadata (think of finding Uganda related material not just in Uganda reports but across all documents, even when "Uganda" as a keyword is not mentioned). It also makes sure that the LLMs dont hallucinate on funding/numeric data where basic mathematics needs to be performed for aggregation purposes. 

## Conclusion

Finding insights from a sea of documents should not be a burden to a Innovation analyst if they are equipped with helpful tools such as a RAG system that can address their questions and cite report sections for its sources. Current approaches rendered the analyst at the mercy of keyword matches only, which was not enough if the reports are cross-talking to each other and topics such as innovation evolution requires tracking portfolios across the years. This work was an effort in the direction of building intelligent systems that brings the relavent chunks and summaries to the analyst instead of the analyst going into them to find their answers. The furture of such intelligent systems is looking bright with LLMs since they excel at learning from the materials they have read across the internet. Just like a newbie analyst, these LLM agents can be given UNICEF reports to ground their reponses. Coding and keyword matching are some skills that these agents also come equipped with. Portfolio managers should see these agents as helpful analyst who also needs some supervision and clear instructions to work on a given problem. Therefore, these systems are not a replacement to UNICEF analysts but tools that bring answers to them in timely manner. They will need fact checking and supervision too, every now and then.
