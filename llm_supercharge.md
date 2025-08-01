


<p align="center">
“We think the future of coding is no coding at all.” - Chris Wanstrath (GitHub co-founder)
</p>

# Chapter 4: Supercharge Migration with LLM

In the previous chapters, we demonstrated how to manually migrate a data pipeline. Now, let's explore how we can leverage AI specifically with a Large Language Model (LLM) along with our declarative REST API Connector to **accelerate the migration process** while preserving a **low but high-quality code surface** that your team can easily maintain by letting an LLM handle most of the boilerplate conversion, while we transform legacy connectors into modern in a fraction of the time.

## Why do this?

* **Accelerate migration to minutes:** LLMs can drastically speed up pipeline migrations that would otherwise take days or weeks. In our context, an AI assistant can be used to understand an API’s documentation or existing connector code and assist in setting up a new connector (the base URL, endpoints, authentication, pagination, etc), which could be tedious and lengthy to completee in a single sitting. [1](https://dlthub.com/docs/dlt-ecosystem/llm-tooling/cursor-restapi)
 
* **Low-code, low-maintenance solution:** The output of an LLM-guided migration is a **declarative, Python-first configuration** rather than hundreds of lines of custom code. This means far less code to maintain. Common logic (auth, pagination, error handling, etc.) is handled by reusable components in the declarative framework, so you’re not rewriting those routines yourself. As a result, there are fewer places for bugs to hide and any improvements apply to all connectors at once. In short, **less custom code = less maintenance overhead**. The low-code approach has been shown to *significantly decrease development effort and bugs while improving maintainability*. Your team can now focus on customizing just the essentials, instead of troubleshooting a large legacy codebase. [2](https://dlthub.com/docs/walkthroughs/deploy-a-pipeline/deploy-with-orchestra)

* **High readability:** The dlt library uses a declarative approach, making data workflows self-documenting and easy to understand. This is achieved through a configuration file that outlines the pipeline's operations, such as endpoints, pagination, and data output. For instance, if a source uses a limit offset pagination scheme, it's explicitly stated in the configuration, not hidden in the code. This clarity allows any engineer to quickly understand the pipeline, simplifying reviews and future modifications.

* **Flexible customization for edge cases:** What if the API is non-standard, using an uncommon pagination method or authentication flow? The declarative framework is extensible. You can always drop down to imperative Python for the unusual cases. For instance, if none of the built-in pagination strategies fit, you can implement a **custom paginator** class and plug it into the declarative config. The same goes for custom auth schemes, rate-limit handling, or data transformations. This means you get the best of both worlds: a low-code base for 90% of the work, with the ability to handle the remaining 10% via custom code when needed. In practice, many APIs follow common patterns (which the connector builder covers), and only the “stranger” APIs require such custom components. [3](https://dlthub.com/docs/general-usage/http/rest-client#implementing-a-custom-paginator)

## LLM-Assisted Migration: How it Works

So how do we actually perform a migration with an LLM in the loop? The process is straightforward and can be summarized in a few steps:

1. **Extract knowledge from the legacy implementation or API docs:** Begin by gathering the source information for the pipeline you want to migrate. This could be the existing connector code or the official API documentation for that source. The goal is to provide the LLM with all the details about endpoints, request/response format, auth requirements, pagination method, and any incremental sync logic.  [4](https://dlthub.com/docs/dlt-ecosystem/llm-tooling/cursor-restapi) 

2. **Let the LLM draft the new declarative config:** Using a prompt or an AI-assisted tool, we ask the LLM to generate a new connector definition using our declarative REST API format. This could be done via our **Connector Builder’s AI Assistant** or a custom script with an LLM backend. The LLM will parse the provided input and produce a configuration (in YAML or Python) that captures the same logic in our framework’s terms. Crucially, the LLM already “knows” the typical structure of a connector: it will propose a base URL, the list of stream endpoints, the request parameters or paths, the method of pagination, the auth scheme, and so on. [5](https://dlthub.com/docs/dlt-ecosystem/llm-tooling/cursor-restapi)

3. **Review and refine the generated config:** While the LLM is powerful, it’s not incapable of error – human oversight is still important. We take the draft configuration and review it carefully. At this stage, we verify details like: Are all important streams included? Did the LLM correctly identify the pagination mechanism? Are the auth parameters and endpoints correct? It’s common to make minor tweaks or fill in any gaps. For example, you might notice that the API requires a specific header that the AI missed, or perhaps the LLM chose a page-size paginator when the API actually uses a next-page cursor token. These are easy fixes to make by editing the config or re-prompting the AI with more context. 

4. **Test the new pipeline:** Once the configuration looks solid, we run it in a test environment. Using the declarative connector with real API credentials, we can attempt a sample sync for each stream. The goal is to validate that the LLM-generated pipeline actually fetches data as expected. Thanks to the standardized nature of our framework, if the config is correct, everything should "just work." We verify that records are being pulled, pagination is working (e.g., it keeps fetching all pages), and incremental sync (if applicable) is updating the state properly. If any issues surface (say, the pagination stops early, or a field is misnamed), we iteratively fix the config or add a small custom component to handle it. After a bit of testing and tweaking, the new pipeline reaches parity with the old one – except now it’s much smaller and easier to understand.

5. **Deploy and monitor:** With a working config in hand, the migration is essentially complete. We replace the old connector with the new declarative one in our pipeline orchestrator. Because the surface area of the new connector is so much slimmer, it’s easier to monitor and trust. We keep an eye on the first few runs to ensure everything is smooth. Over time, maintenance should also be simpler: if the source API changes (new fields or an endpoint URL change), updating our declarative config is straightforward. The team can modify a line or two in the YAML rather than refactoring a large codebase.

Throughout this process, the LLM serves as an **accelerator**. It automates the translation of legacy logic into our modern config format, handling the repetitive work of mapping fields and endpoints. However, we still need to apply our expertise to guide the AI providing correct context and prompts and to verify the output. This **human-in-the-loop** approach ensures we get the best results.

---

## Benefits Recap: A Better Pipeline, Faster

By using AI-powered migration in combination with the declarative REST API connector, we achieve several important benefits:

* **Minutes to migrate, not months:** What used to be a painful manual migration is now dramatically accelerated. Teams have reported huge time savings by using LLMs for code transformation tasks – in one case, an AI system automated 74% of code edits and cut developer effort by 50%. In our scenario, even if the LLM doesn’t get everything perfect on the first try, it handles the bulk of the work instantly, allowing the migration to be completed very quickly. This speed means you can migrate more pipelines and keep up with deprecations or upgrades proactively, rather than postponing them due to lack of time.

* **Lean, high-quality connector surface:** The final migrated pipeline is defined in a concise configuration that uses battle-tested components. There is far less custom code. This **low-code outcome** is easier to maintain and less prone to bugs by design. As noted earlier, eliminating duplicate code logic means any bug fix or improvement in a common component benefits all connectors and only needs to be done once. Your team can maintain hundreds of such connectors consistently, rather than wrestling with each one’s idiosyncratic code. The migration process itself can improve quality too – it’s an opportunity to **audit and clean up** the pipeline’s behavior. Many legacy connectors have grown complex over time; converting them with an AI often reveals simplifications (e.g. removing outdated fields or using a better incremental strategy) that you can incorporate, resulting in a cleaner pipeline than the original.

* **Clarity and transparency:** A declarative pipeline config reads almost like documentation. This high readability is a boon for long-term usage. New team members can onboard faster by reading the YAML/Python config to understand the data flow. Debugging is also easier – since each piece of logic (request structure, response parsing, etc.) is declaratively specified, you can pinpoint issues (like a wrong field path or missing parameter) at a glance. The manifest doubles as a reference for how the integration works. In fact, we can even **analyze these configs at scale**; for example, we could quickly query how many connectors use a certain pagination type or auth method, because it’s all declarative metadata. Such introspection is much harder when logic is scattered across imperative code.

* **Extensibility when needed:** Despite being low-code, our framework is not a black box – you retain full power to extend and customize. If the LLM-generated base config isn’t handling a quirky aspect of the API, you can insert a small Python component to address it. We’ve made sure that every part of the connector (authentication, requests, error handling, pagination, record parsing, etc.) can be overridden with custom code if necessary. For example, for an API that uses a truly one-off pagination scheme, you might write a `MyCustomPaginationStrategy` class implementing our interface, and just reference it in the config. The key point is that **these cases are the exception rather than the norm** – most APIs will work with out-of-the-box components. But knowing you have this escape hatch gives confidence that the declarative approach can handle even the tricky sources. It also means an LLM can get you 90% of the way there, and you or another engineer can handle the last 10% with a few lines of code if needed.

In summary, combining an LLM with a declarative connector framework supercharges your migration efforts. You get the **best of AI and human expertise**: the AI rapidly converts legacy logic into a proposed config, and you fine-tune and verify it. The end result is a modern pipeline that is easier to read, maintain, and extend. By turning legacy code into a Python-first configuration, you make your data platform more robust and adaptable to change. Migrations no longer need to be daunting, drawn-out projects – with this approach, you can perform them quickly and with confidence, keeping your pipelines up-to-date and your team focused on higher-value tasks rather than plumbing.

With the migration accelerated and the pipeline surface greatly simplified, you’ve not only saved time but also set your team up for long-term success. The next time an API changes or a new source needs onboarding, you can reach for this LLM-assisted, declarative method and tackle the task in record time. This concludes our exploration of supercharging pipeline migration with AI – next, we will look at **\[the next chapter topic, if applicable]**, continuing to build on these modern data engineering practices.


## Definition of Done
Your LLM-assisted migration is complete when:

1. Pipeline runs successfully against Binance endpoints.

2. Pagination works—all data pages are fetched correctly.

3. Incremental logic is validated on multiple runs.

4. The generated config is camera-ready for code review: clean, documented, minimal.

5. Pipeline is deployed, monitored, and approved by stakeholders.

Once in production, feeding new endpoints (e.g. account balances or trade history) is a matter of updating your declarative config—not rewriting code.

## Next Step: Why This Matters
Following Chapters 1–3, we've built your migration rationale, scaffolded a new pipeline, and converted legacy logic to dlt. This LLM-assisted approach is the superpower that lets you scale migrations:

From one pipeline to dozens, without rewriting

From java/python scripts to maintainable dlt config

From manual pagination to AI-powered standardization

In Chapter 5, we'll dive deeper into backfilling historical data and stitching legacy systems—so you get full coverage without disrupting production.





