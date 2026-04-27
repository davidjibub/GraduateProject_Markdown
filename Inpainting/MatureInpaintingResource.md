# Inpainting
## TumorSynth
Mar 27 2026
### Article
https://surfer.nmr.mgh.harvard.edu/fswiki/TumorSynth?utm_source=chatgpt.com
https://pubs.rsna.org/doi/10.1148/rycan.250222

TumorSynth 路线：目标是直接在含瘤图像上联合输出健康组织标签和肿瘤标签，并通过第二阶段细分肿瘤内部结构；官方实现基于 U-Net / nnU-Net 级联分割，而非 diffusion-based image synthesis。

### Docker
https://github.com/fprados/TumorSynth/tree/main/