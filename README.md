# gridfm-datakit-v1: A Python Library for Scalable and Realistic Power Flow and Optimal Power Flow Data Generation

These are step-by-step guides for reproducing the results of [gridfm-datakit-v1: A Python Library for Scalable and Realistic Power Flow and Optimal Power Flow Data Generation](https://arxiv.org/abs/2512.14658) (arXiv:2512.14658).

This covers the entropy spider plots, the feature violin plots, the branch-flow entropy barplot, and the branch-loading histograms.

## 1. Install

Python 3.10–3.12. Branch [genco-paper-repro](https://github.com/gridfm/gridfm-datakit/tree/genco-paper-repro).

```bash
git clone -b genco-paper-repro https://github.com/gridfm/gridfm-datakit.git
cd gridfm-datakit
pip install -e '.[dev,test]'
pip install seaborn
export PYTHONPATH=$PWD
```

Run the later commands from that repository root.

## 2. Data

The sampled parquet used in the figures is already saved in [gridfm/reproducibility-datakit-technical-report](https://huggingface.co/datasets/gridfm/reproducibility-datakit-technical-report).

Six datasets on `case118_ieee` sit at the repo root. Each is downsampled to 10,000 scenarios. Every file is `<prefix>_bus_data.parquet` and `<prefix>_gen_data.parquet`. The two PF datasets also have `<prefix>_branch_data.parquet`.

| Prefix | Mode | Library |
| --- | --- | --- |
| `gridfm_datakit_pf` | PF | gridfm-datakit, `mode: pf` |
| `gridfm_datakit_opf` | OPF | gridfm-datakit, `mode: opf` |
| `pfdelta` | PF | PFΔ |
| `opfdata` | OPF | OPFData |
| `pglearn` | OPF | PGLearn |
| `opflearn` | OPF | OPF-Learn |

Download those sampled files only. `--exclude "full/*"` leaves out the larger tables and keeps the download to about 1 GB. The scripts read `scripts/datakit_report/dataset_sampled/` by default. Pass `--data-dir` to use another path.

```bash
hf download gridfm/reproducibility-datakit-technical-report \
  --repo-type dataset \
  --exclude "full/*" \
  --local-dir scripts/datakit_report/dataset_sampled
```

To generate the figures from these saved results, jump to [Figures and tables](#4-figures-and-tables).

## 3. Regenerating the sampled data from scratch

The parquet in the previous section is a sample of larger datasets. Those tables are in the same Hugging Face repo, under `full/`. You do not need them to rebuild the figures. That tree is about 15 GB.

```bash
hf download gridfm/reproducibility-datakit-technical-report \
  --repo-type dataset \
  --include "full/*" \
  --local-dir scripts/datakit_report/dataset_full
```

| Folder | Scenarios |
| --- | --- |
| `full/gridfm_datakit_pf/` | 199,207 |
| `full/gridfm_datakit_opf/` | 195,753 |
| `full/opfdata/` | 300,000 |
| `full/pglearn/` | 96,852 |
| `full/opflearn/` | 10,000 |
| `full/pfdelta/` | 29,000 |

### Sampling the snapshots

[prepare_datasets.py](https://github.com/gridfm/gridfm-datakit/blob/genco-paper-repro/scripts/datakit_report/prepare_datasets.py) downsamples every dataset to the smallest scenario count.

```bash
python scripts/datakit_report/prepare_datasets.py --base-path /path/to/raw/datasets
```

The draw is random, so a new run does not reproduce the published figures. Use the Hugging Face snapshot for the report figures.

gridfm-datakit inputs come from [scripts/datakit_report/configs/](https://github.com/gridfm/gridfm-datakit/tree/genco-paper-repro/scripts/datakit_report/configs):

```bash
gridfm_datakit generate scripts/datakit_report/configs/case118_ieee_pf.yaml
```

The other libraries have to be converted to this parquet schema first.

| Library | Converter |
| --- | --- |
| PFΔ | [pfdelta/batch_convert_pfdelta.py](https://github.com/gridfm/gridfm-datakit/blob/genco-paper-repro/pfdelta/batch_convert_pfdelta.py) |
| OPFData | [opf_data/batch_convert.py](https://github.com/gridfm/gridfm-datakit/blob/genco-paper-repro/opf_data/batch_convert.py) |
| PGLearn | [pg_learn_conversion.py](https://github.com/gridfm/gridfm-datakit/blob/genco-paper-repro/scripts/datakit_report/pg_learn_conversion.py). Set `PGLEARN_DIR`. |
| OPF-Learn | [opf_learn_conversion.py](https://github.com/gridfm/gridfm-datakit/blob/genco-paper-repro/scripts/datakit_report/opf_learn_conversion.py). Set `OPFLEARN_DIR`. |

## 4. Figures and tables

From the datakit repo root. Scripts are in [scripts/datakit_report/](https://github.com/gridfm/gridfm-datakit/tree/genco-paper-repro/scripts/datakit_report).

```bash
OUT=scripts/datakit_report/out

python scripts/datakit_report/plot_spider.py --mode pf --metric entropy --output-dir $OUT
python scripts/datakit_report/plot_spider.py --mode opf --metric entropy --output-dir $OUT

python scripts/datakit_report/plot_violin.py --mode pf --output-dir $OUT
python scripts/datakit_report/plot_violin.py --mode opf --output-dir $OUT

python scripts/datakit_report/plot_bar_branch.py --metric entropy --output-dir $OUT

python scripts/datakit_report/plot_branch_loading.py --output-dir $OUT
```

That writes 17 PDFs into `scripts/datakit_report/out/`.

| Script | Output |
| --- | --- |
| `plot_spider.py` | `spider_plot_entropy_{pf,opf}.pdf` |
| `plot_violin.py` | `{Pd,Qd,Pg,Qg,Vm,Va}_violin_{pf,opf}.pdf` |
| `plot_bar_branch.py` | `barplot_branch_entropy_pf.pdf` |
| `plot_branch_loading.py` | `branch_loading_{datakit,pfdelta}.pdf` |

## Notes

- Violin panels use the 10 buses listed in `PINNED_BUSES` in [plot_violin.py](https://github.com/gridfm/gridfm-datakit/blob/genco-paper-repro/scripts/datakit_report/plot_violin.py). Pass `--no-pin` to sample buses instead.
- [plot_branch_loading.py](https://github.com/gridfm/gridfm-datakit/blob/genco-paper-repro/scripts/datakit_report/plot_branch_loading.py) sets the matplotlib backend to `Agg` before importing pyplot. Leave that import order as it is.
- [plot_spider_branch.py](https://github.com/gridfm/gridfm-datakit/blob/genco-paper-repro/scripts/datakit_report/plot_spider_branch.py) and [plot_violin_branch.py](https://github.com/gridfm/gridfm-datakit/blob/genco-paper-repro/scripts/datakit_report/plot_violin_branch.py) are not used for the report figures. `plot_spider_branch.py` writes `barplot_branch_{metric}_pf.pdf`, the same name as `plot_bar_branch.py`. Point it at a separate `--output-dir`.
