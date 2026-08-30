[30th August, 2026 Log]

The methodology I understand is in reference to echoJEPA's working,

EchoJEPA claims itself to be a self-supervised foundation model for echocardiography:![[echo.png]]

So what it does in short is:

1. Input consists of short echo video clips (built on V-JEPA2).
2. Masking is done to hide several chunks of the video across both space and time (spatiotemporal masking, well thats what they call it)
3. Context Encoder acts as a vision transformer which looks only at the _unmasked_ parts.
4. Predict the hidden parts in (buzzword in ... 3 ... 2 ... 1) **latent space** and not pixel-level, a small predictor network tries to guess what the _embeddings_  of the masked chunks should be. The answer comes from a target encoder  which is based on a EMA (exponential moving average) and a copy of the context encoder which runs on the full clip.
5. The Loss here is the difference between predicted embeddings and target embeddings, measured in feature space. Crucially, it NEVER reconstructs pixels.

Ultrasound is dominated by speckle, acoustic shadows, and intensity artifacts that vary between scans (whatever this bs means) and have nothing to do with heart anatomy. 

A pixel-reconstruction model wastes capacity redrawing that noise so predicting in latent space lets the model throw the noise away and keep only the meaningful structure.

Then the encoder is **frozen** since it isolates representation quality & generally allows more stability across small datasets, and a small attentive-probe head is trained on top for clinical tasks ejection fraction, view classification, and so on and so forth.

Finally EchoJEPA achieves
- Left ventricular ejection fraction (LVEF) estimation which is a _regression_ task. LVEF is the percentage of blood the heart pumps out per beat and low LVEF signals heart failure. 
- Right ventricular systolic pressure (RVSP) estimation is also another regression task.
- View classification (self-explanatory) identifying which echocardiographic view/angle a clip shows (apical-4-chamber, parasternal, etc.).

They evaluated on EchoNet-Dynamic dataset for all their benchmarking ideas.

So ...

The TCGA-UT dataset is composed of images where the removed tissue is first (1) fixed in formalin and embedded in paraffin wax, (2) sliced into ultra-thin sections a few microns thick, (3) stained with **H&E** hematoxylin turns cell nuclei blue/purple, eosin turns cytoplasm and connective tissue pink, (4) mounted on a glass slide, and (5) scanned by a whole-slide scanner at high magnification into a single gigapixel digital image.

it's precisely 1,608,060 small tiles, each 256×256 pixels, cropped from _tumor regions_ that two pathologists selected, spanning 32 cancer types (breast, colon, brain, prostate & most importantly lung [most relevant to us])

So the correspondence to echoJEPA is in the ideation that the **stain and scanner variation.** If you took the exact slide in an image to a different hospital and ran it through a different scanner, the glands and nuclei would be biologically identical, but the pinks and purples would shift, the brightness would change, the sharpness would differ.

That shift here should act as the **speckle**: non-biological, acquisition-dependent, and something JEPA should ignore. 

For cancerJEPA:
1. Patch-level (tile) classification

- **BACH**: 4-class breast cancer
- **CRC (NCT-CRC-100K)** : 9-class colorectal tissue
- **MHIST**:binary colorectal polyp
- **PatchCamelyon (PCam)**: binary breast tumor (patches derived from CAMELYON16)
- Extended set also includes **BRACS, BreakHis, Gleason grading, and UniToPatho**

 _Metric:_ balanced accuracy (binary) / multiclass accuracy, averaged over 5 runs.

2. Nuclei segmentation

- **CoNSeP** :colorectal nuclei segmentation & phenotypes
- **MoNuSAC** : multi-organ nuclei segmentation

_Metric:_ Dice score (foreground).

3. Slide-level (WSI) classification

- **CAMELYON16** is a breast metastasis detection 
- **PANDA** is a prostate Gleason/ISUP grading (4-class, ~5,158 slides)

 _Metric:_ AUROC / balanced accuracy for CAMELYON16; accuracy or quadratic-weighted kappa for PANDA
 
For the downstream tasks we could probably do tumor or metastasis detection, gleason grading & maybe tumour classification.

 Stuff to read: https://arxiv.org/pdf/2212.04690 which seems relevant 