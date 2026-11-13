# Technical Articles
> Repository: [GitHub](https://github.com/solomonxu/ai-maas-docs) · [Gitee](https://gitee.com/solomonxu/ai-maas-docs)

[English](./README.en.md) | [简体中文](./README.md)

The Building OpenMaas engineering series. Charts and test data are local baselines, not production SLAs.

## Articles

| Article | Description |
|---|---|
| [Designing a Recoverable, Cost-Aware, Automatable Training-to-Inference Pipeline](./pipeline-technical-article-en.md) | Run model, recovery, preflight, cost, and automation |
| [把模型训推做成一条可恢复、可计费、可自动化的流水线](./pipeline-technical-article.md) | Chinese version |
| [Four Resources on One GPU: GPU Resource Governance for Training and Inference Platforms](./gpu-resource-governance-article-en.md) | VRAM, compute time, leases, residency, QoS, and eviction |
| [一块 GPU 上的四种资源：训推平台的 GPU 资源治理](./gpu-resource-governance-article.md) | Chinese version |

## Notes

- Diagram labels are in English; captions and body text are localized for reuse;
- Source diagrams live in [`assets/`](./assets/);
- Only aggregate charts are public; raw JSON is not included;
- "Cost-aware" means cost can be calculated, attributed, and explained, not that OpenMaas is a complete billing system.
