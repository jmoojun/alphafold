![header](imgs/header.jpg)

# AlphaFold

This package provides an implementation of the inference pipeline of AlphaFold
v2.0. This is a completely new model that was entered in CASP14 and published in
Nature. For simplicity, we refer to this model as AlphaFold throughout the rest
of this document.

Any publication that discloses findings arising from using this source code or
the model parameters should [cite](#citing-this-work) the
[AlphaFold paper](https://doi.org/10.1038/s41586-021-03819-2).

![CASP14 predictions](imgs/casp14_predictions.gif)

Seoklab version of AlphaFold has few changes:

- Major changes
  - No docker, no system libraries (please refer to [kalininalab/alphafold_non_docker](https://github.com/kalininalab/alphafold_non_docker) repository and [install.sh](install.sh)).
  - Can use multiple GPUs for inference.
  - Skip relaxation step, as it takes very long time (~30 min per model) for preprocessing.
  - Add environment variables `ALPHAFOLD_HOME` and `ALPHAFOLD_CONDA_PREFIX` for dynamic path resolving.
- Minor changes
  - Add runner script [`alphafold`](bin/alphafold).
  - Support "resuming"; this version will automatically try to use the previous results, if they exists. You can force everything to run again by passing `--overwrite` flag from the command line.
  - Make command line interface more user-friendly.
  - Code refactoring.

## First time setup

Clone this repository, then run `./install.sh`. The script requires `wget` to run. Few variables could change the behavior of the script, namely:

- `$CONDA_PREFIX`: The path for the newly-installed miniconda. Defaults to `/opt/conda` (for system-wide installations).
- `$SUDO`: Either `y` or `n`. Defaults to `y`. If set to `y`, then the script will try to install AlphaFold system-wide, invoking `sudo` a few times. **Please be careful for running this script in SUDO mode.** Even though this script has been tested a few times, it is **NOT** fully tested for all types of Linux distros. (Currently tested in Ubuntu Server 16.04 LTS and Ubuntu Server 20.04 LTS)

### Genetic databases

This step requires `rsync` and `aria2c` to be installed on your machine.

AlphaFold needs multiple genetic (sequence) databases to run:

*   [UniRef90](https://www.uniprot.org/help/uniref),
*   [MGnify](https://www.ebi.ac.uk/metagenomics/),
*   [BFD](https://bfd.mmseqs.com/),
*   [Uniclust30](https://uniclust.mmseqs.com/),
*   [PDB70](http://wwwuser.gwdg.de/~compbiol/data/hhsuite/databases/hhsuite_dbs/),
*   [PDB](https://www.rcsb.org/) (structures in the mmCIF format).

We provide a script `scripts/download_all_data.sh` that can be used to download
and set up all of these databases. This should take 8–12 hours. **The data should be downloaded into `$ALPHAFOLD_HOME/data`.** If you have chosen the other directory for the data to live, then it must be explicitly passed to the downloader script as a command line argument. The script will then automatically create a symlink pointing to the target directory at `$ALPHAFOLD_HOME/data`.

If such behavior is not desired or another database is being used, then the data directory could be explicitly passed as arguments, when invoking the `alphafold` script. (Please refer to the [next section](#running-alphafold) for more details.)

:ledger: **Note: The total download size is around 428 GB and the total size
when unzipped is 2.2 TB. Please make sure you have a large enough hard drive
space, bandwidth and time to download.**

This script will also download the model parameter files. Once the script has
finished, you should have the following directory structure:

```txt
$ALPHAFOLD_HOME/
    data/                             # Total: ~ 2.2 TB (download: 428 GB)
        bfd/                                   # ~ 1.8 TB (download: 271.6 GB)
            # 6 files.
        mgnify/                                # ~ 64 GB (download: 32.9 GB)
            mgy_clusters.fa
        params/                                # ~ 3.5 GB (download: 3.5 GB)
            # 5 CASP14 models,
            # 5 pTM models,
            # LICENSE,
            # = 11 files.
        pdb70/                                 # ~ 56 GB (download: 19.5 GB)
            # 9 files.
        pdb_mmcif/                             # ~ 206 GB (download: 46 GB)
            mmcif_files/
                # About 180,000 .cif files.
            obsolete.dat
        uniclust30/                            # ~ 87 GB (download: 24.9 GB)
            uniclust30_2018_08/
                # 13 files.
        uniref90/                              # ~ 59 GB (download: 29.7 GB)
            uniref90.fasta
```

### Model parameters

While the AlphaFold code is licensed under the Apache 2.0 License, the AlphaFold
parameters are made available for non-commercial use only under the terms of the
CC BY-NC 4.0 license. Please see the [Disclaimer](#license-and-disclaimer) below
for more detail.

The AlphaFold parameters are available from
https://storage.googleapis.com/alphafold/alphafold_params_2021-07-14.tar, and
are downloaded as part of the `scripts/download_all_data.sh` script. This script
will download parameters for:

*   5 models which were used during CASP14, and were extensively validated for
    structure prediction quality (see Jumper et al. 2021, Suppl. Methods 1.12
    for details).
*   5 pTM models, which were fine-tuned to produce pTM (predicted TM-score) and
    predicted aligned error values alongside their structure predictions (see
    Jumper et al. 2021, Suppl. Methods 1.9.7 for details).

## Running AlphaFold

Invoke the runner script `alphafold` with the fasta paths as arguments. Full configurations is as followings.

```txt
usage: alphafold [-h] [--helpfull] [--output_dir OUTPUT_DIR]
                 [--model_cnt MODEL_CNT] [--nproc NPROC]
                 [--max_template_date MAX_TEMPLATE_DATE]
                 [--ensemble ENSEMBLE] [--model_type MODEL_TYPE]
                 [--benchmark] [--data_dir DATA_DIR]
                 [--jackhmmer_binary_path JACKHMMER_BINARY_PATH]
                 [--hhblits_binary_path HHBLITS_BINARY_PATH]
                 [--hhsearch_binary_path HHSEARCH_BINARY_PATH]
                 [--kalign_binary_path KALIGN_BINARY_PATH]
                 [--uniref90_database_path UNIREF90_DATABASE_PATH]
                 [--mgnify_database_path MGNIFY_DATABASE_PATH]
                 [--bfd_database_path BFD_DATABASE_PATH]
                 [--uniclust30_database_path UNICLUST30_DATABASE_PATH]
                 [--pdb70_database_path PDB70_DATABASE_PATH]
                 [--template_mmcif_dir TEMPLATE_MMCIF_DIR]
                 [--obsolete_pdbs_path OBSOLETE_PDBS_PATH]
                 [--random_seed_seed RANDOM_SEED_SEED] [--overwrite]
                 fasta_paths [fasta_paths ...]

positional arguments:
  fasta_paths           Paths to FASTA files, each containing one sequence.
                        All FASTA paths must have a unique basename as the
                        basename is used to name the output directories for
                        each prediction.

optional arguments:
  -h, --help            show this help message and exit
  --helpfull            show full help message and exit
  --output_dir OUTPUT_DIR
                        Path to a directory that will store the results.
  --model_cnt MODEL_CNT
                        Counts of models to use. Note that AlphaFold provides
                        5 pretrained models, so setting the count other than 5
                        is either redundant or insufficient configuration.
  --nproc NPROC         Maximum cpu count to use. Note that the actual cpu
                        load might be different than the configured value.
  --max_template_date MAX_TEMPLATE_DATE
                        Maximum template release date to consider(ISO-8601
                        format - i.e. YYYY-MM-DD). Important if folding
                        historical test sets.
  --ensemble ENSEMBLE   Choose ensemble count: note that AlphaFold recommends
                        1 ("full_dbs"), and the casp model have used 8 model
                        ensemblings ("casp14").
  --model_type MODEL_TYPE
                        <normal|ptm>: Choose model type to use - the casp14
                        equivalent model (normal), or fined-tunded pTM models
                        (ptm).
  --benchmark, --nobenchmark
                        Run multiple JAX model evaluations to obtain a timing
                        that excludes the compilation time, which should be
                        more indicative of the time required for inferencing
                        many proteins.
  --data_dir DATA_DIR   Path to directory of supporting data.
  --jackhmmer_binary_path JACKHMMER_BINARY_PATH
                        Path to the JackHMMER executable.
  --hhblits_binary_path HHBLITS_BINARY_PATH
                        Path to the HHblits executable.
  --hhsearch_binary_path HHSEARCH_BINARY_PATH
                        Path to the HHsearch executable.
  --kalign_binary_path KALIGN_BINARY_PATH
                        Path to the Kalign executable.
  --uniref90_database_path UNIREF90_DATABASE_PATH
                        Path to the Uniref90 database for use by JackHMMER.
  --mgnify_database_path MGNIFY_DATABASE_PATH
                        Path to the MGnify database for use by JackHMMER.
  --bfd_database_path BFD_DATABASE_PATH
                        Path to the BFD database for use by HHblits.
  --uniclust30_database_path UNICLUST30_DATABASE_PATH
                        Path to the Uniclust30 database for use by HHblits.
  --pdb70_database_path PDB70_DATABASE_PATH
                        Path to the PDB70 database for use by HHsearch.
  --template_mmcif_dir TEMPLATE_MMCIF_DIR
                        Path to a directory with template mmCIF structures,
                        each named <pdb_id>.cif
  --obsolete_pdbs_path OBSOLETE_PDBS_PATH
                        Path to file containing a mapping from obsolete PDB
                        IDs to the PDB IDs of their replacements.
  --random_seed_seed RANDOM_SEED_SEED
                        The random seed for the random seed for the data
                        pipeline. By default, this is randomly generated. Note
                        that even if this is set,Alphafold may still not be
                        deterministic, because processes like GPU inference
                        are nondeterministic.
  --overwrite, --nooverwrite
                        Whether to re-build the features, even if the result
                        exists in the target directories.
```

### AlphaFold output

The outputs will be **in the current directory** for the default settings. They include the computed MSAs,
unrelaxed structures, relaxed structures, ranked structures, raw model outputs,
prediction metadata, and section timings. The directory will have
the following structure:

```
./
    features.pkl
    ranked_{0,1,2,3,4,...}.pdb
    ranking_debug.json
    relaxed_model_{1,2,3,4,5,...}.pdb
    result_model_{1,2,3,4,5,...}.pkl
    timings.json
    unrelaxed_model_{1,2,3,4,5,...}.pdb
    msas/
        bfd_uniclust_hits.a3m
        mgnify_hits.sto
        uniref90_hits.a3m
        uniref90_hits.sto
```

The contents of each output file are as follows:

*   `features.pkl` – A `pickle` file containing the input feature Numpy arrays
    used by the models to produce the structures.
*   `unrelaxed_model_*.pdb` – A PDB format text file containing the predicted
    structure, exactly as outputted by the model.
*   `relaxed_model_*.pdb` – A PDB format text file containing the predicted
    structure, after performing an Amber relaxation procedure on the unrelaxed
    structure prediction, see Jumper et al. 2021, Suppl. Methods 1.8.6 for
    details.
*   `ranked_*.pdb` – A PDB format text file containing the relaxed predicted
    structures, after reordering by model confidence. Here `ranked_0.pdb` should
    contain the prediction with the highest confidence, and `ranked_4.pdb` the
    prediction with the lowest confidence. To rank model confidence, we use
    predicted LDDT (pLDDT), see Jumper et al. 2021, Suppl. Methods 1.9.6 for
    details.
*   `ranking_debug.json` – A JSON format text file containing the pLDDT values
    used to perform the model ranking, and a mapping back to the original model
    names.
*   `timings.json` – A JSON format text file containing the times taken to run
    each section of the AlphaFold pipeline.
*   `msas/` - A directory containing the files describing the various genetic
    tool hits that were used to construct the input MSA.
*   `result_model_*.pkl` – A `pickle` file containing a nested dictionary of the
    various Numpy arrays directly produced by the model. In addition to the
    output of the structure module, this includes auxiliary outputs such as
    distograms and pLDDT scores. If using the pTM models then the pTM logits
    will also be contained in this file.

This code has been tested to match mean top-1 accuracy on a CASP14 test set with
pLDDT ranking over 5 model predictions (some CASP targets were run with earlier
versions of AlphaFold and some had manual interventions; see our forthcoming
publication for details). Some targets such as T1064 may also have high
individual run variance over random seeds.

## Inferencing many proteins

The provided inference script is optimized for predicting the structure of a
single protein, and it will compile the neural network to be specialized to
exactly the size of the sequence, MSA, and templates. For large proteins, the
compile time is a negligible fraction of the runtime, but it may become more
significant for small proteins or if the multi-sequence alignments are already
precomputed. In the bulk inference case, it may make sense to use our
`make_fixed_size` function to pad the inputs to a uniform size, thereby reducing
the number of compilations required.

We do not provide a bulk inference script, but it should be straightforward to
develop on top of the `RunModel.predict` method with a parallel system for
precomputing multi-sequence alignments. Alternatively, this script can be run
repeatedly with only moderate overhead.

## Note on reproducibility

AlphaFold's output for a small number of proteins has high inter-run variance,
and may be affected by changes in the input data. The CASP14 target T1064 is a
notable example; the large number of SARS-CoV-2-related sequences recently
deposited changes its MSA significantly. This variability is somewhat mitigated
by the model selection process; running 5 models and taking the most confident.

To reproduce the results of our CASP14 system as closely as possible you must
use the same database versions we used in CASP. These may not match the default
versions downloaded by our scripts.

For genetics:

*   UniRef90:
    [v2020_01](https://ftp.uniprot.org/pub/databases/uniprot/previous_releases/release-2020_01/uniref/)
*   MGnify:
    [v2018_12](http://ftp.ebi.ac.uk/pub/databases/metagenomics/peptide_database/2018_12/)
*   Uniclust30: [v2018_08](http://wwwuser.gwdg.de/~compbiol/uniclust/2018_08/)
*   BFD: [only version available](https://bfd.mmseqs.com/)

For templates:

*   PDB: (downloaded 2020-05-14)
*   PDB70: (downloaded 2020-05-13)

An alternative for templates is to use the latest PDB and PDB70, but pass the
flag `--max_template_date=2020-05-14`, which restricts templates only to
structures that were available at the start of CASP14.

## Citing this work

If you use the code or data in this package, please cite:

```tex
@Article{AlphaFold2021,
  author  = {Jumper, John and Evans, Richard and Pritzel, Alexander and Green, Tim and Figurnov, Michael and Ronneberger, Olaf and Tunyasuvunakool, Kathryn and Bates, Russ and {\v{Z}}{\'\i}dek, Augustin and Potapenko, Anna and Bridgland, Alex and Meyer, Clemens and Kohl, Simon A A and Ballard, Andrew J and Cowie, Andrew and Romera-Paredes, Bernardino and Nikolov, Stanislav and Jain, Rishub and Adler, Jonas and Back, Trevor and Petersen, Stig and Reiman, David and Clancy, Ellen and Zielinski, Michal and Steinegger, Martin and Pacholska, Michalina and Berghammer, Tamas and Bodenstein, Sebastian and Silver, David and Vinyals, Oriol and Senior, Andrew W and Kavukcuoglu, Koray and Kohli, Pushmeet and Hassabis, Demis},
  journal = {Nature},
  title   = {Highly accurate protein structure prediction with {AlphaFold}},
  year    = {2021},
  doi     = {10.1038/s41586-021-03819-2},
  note    = {(Accelerated article preview)},
}
```

## Acknowledgements

AlphaFold communicates with and/or references the following separate libraries
and packages:

*   [Abseil](https://github.com/abseil/abseil-py)
*   [Biopython](https://biopython.org)
*   [Chex](https://github.com/deepmind/chex)
*   [Docker](https://www.docker.com)
*   [HH Suite](https://github.com/soedinglab/hh-suite)
*   [HMMER Suite](http://eddylab.org/software/hmmer)
*   [Haiku](https://github.com/deepmind/dm-haiku)
*   [Immutabledict](https://github.com/corenting/immutabledict)
*   [JAX](https://github.com/google/jax/)
*   [Kalign](https://msa.sbc.su.se/cgi-bin/msa.cgi)
*   [ML Collections](https://github.com/google/ml_collections)
*   [NumPy](https://numpy.org)
*   [OpenMM](https://github.com/openmm/openmm)
*   [OpenStructure](https://openstructure.org)
*   [SciPy](https://scipy.org)
*   [Sonnet](https://github.com/deepmind/sonnet)
*   [TensorFlow](https://github.com/tensorflow/tensorflow)
*   [Tree](https://github.com/deepmind/tree)

Seoklab version makes use of one extra libary:

* [Joblib](https://github.com/joblib/joblib)

We thank all their contributors and maintainers!

## License and Disclaimer

This is not an officially supported Google product.

Copyright 2021 DeepMind Technologies Limited.

### AlphaFold Code License

Licensed under the Apache License, Version 2.0 (the "License"); you may not use
this file except in compliance with the License. You may obtain a copy of the
License at https://www.apache.org/licenses/LICENSE-2.0.

Unless required by applicable law or agreed to in writing, software distributed
under the License is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR
CONDITIONS OF ANY KIND, either express or implied. See the License for the
specific language governing permissions and limitations under the License.

### Model Parameters License

The AlphaFold parameters are made available for non-commercial use only, under
the terms of the Creative Commons Attribution-NonCommercial 4.0 International
(CC BY-NC 4.0) license. You can find details at:
https://creativecommons.org/licenses/by-nc/4.0/legalcode

### Third-party software

Use of the third-party software, libraries or code referred to in the
[Acknowledgements](#acknowledgements) section above may be governed by separate
terms and conditions or license provisions. Your use of the third-party
software, libraries or code is subject to any such terms and you should check
that you can comply with any applicable restrictions or terms and conditions
before use.

### Mirrored Databases

The following databases have been mirrored by DeepMind, and are available with reference to the following:

*   [BFD](https://bfd.mmseqs.com/) (unmodified), by Steinegger M. and Söding J., available under a [Creative Commons Attribution-ShareAlike 4.0 International License](http://creativecommons.org/licenses/by-sa/4.0/).
