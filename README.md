# Improving CEMA using Correlation Optimization

This repository contains the code and datasets required to reproduce the results from the paper [Improving CEMA using Correlation Optimization](https://tches.iacr.org/index.php/TCHES/article/view/7332).

# Datasets

The following datasets are provided, which can be downloaded from the links below:

- [`ASCAD`](https://github.com/ANSSI-FR/ASCAD): Datasets provided by Prouff et al. that were used for the evaluation of an AES algorithm with a masking countermeasure (Figures 4 - 9). See the author's paper for more information about the dataset.
- [`em-corr-arduino-pre`](https://drive.google.com/file/d/1pGYh-C6cSoVUwssfbgTryoc2Fbk9o4AB/view?usp=sharing): Preprocessed (aligned and filtered) Arduino Duemilanove AES EM leakage training set.
- [`em-cpa-arduino-pre`](https://drive.google.com/file/d/1Qhf-yuHspw66dnkMBFIYsr6fls2HyF2_/view?usp=sharing): Preprocessed (aligned and filtered) Arduino Duemilanove AES EM leakage test set.
- [`em-corr-arduino`](https://drive.google.com/file/d/1Vdi0EmfgIqEQvhcL2cBtn76PfLb2azaw/view?usp=sharing): Arduino Duemilanove AES EM leakage training set.
- [`em-cpa-arduino`](https://drive.google.com/file/d/1qB3KhwvexCe-s9NbIzR9QYp9VHkhVpVb/view?usp=sharing): Arduino Duemilanove AES EM leakage test set.
- Custom "Arduino Duemilanove AES" dataset: see below

## Arduino Duemilanove AES dataset
The Arduino Duemilanove AES dataset comprises a training set and test set, named `em-corr-arduino` and `em-cpa-arduino` respectively. Both datasets are formatted according to the [`ChipWhisperer`](https://newae.com/tools/chipwhisperer/) format:

- Files ending in `_traces.npy` contain a `[num_traces, num_samples]` Numpy array of a time-series of magnitudes, captured with a USRP centered at 71 MHz and with a sample rate of 8 Ms/s. The EM probe was placed over the VCC and GND pins.
- Files ending in `_textin.npy` contain a `[num_traces, num_plaintext_bytes]` Numpy array of the plaintext used during each trace of the AES encryption.
- Files ending in `_knownkey.npy` contain a `[num_traces, num_key_bytes]` Numpy array of the key used during each trace of the AES encryption.

The `em-corr-arduino` dataset contains traces where both the used key and plaintext are random. For `em-cpa-arduino`, the used key is static (`00 01 02 ... 15`) and the plaintext is random.

Each trace in the datasets has a variable number of samples, depending on the execution time of the AES algortihm. The traces are neither aligned nor filtered. Using the `filter` and `align` commands in EMMA (see the 'Code' section below), you can perform these actions as needed by some attacks. Alternatively, you can use the `em-corr-arduino-pre` and `em-cpa-arduino-pre` datasets, where the traces have already been filtered, aligned, and windowed to 1,300 samples per trace. Note that these datasets have not been used in the paper, yet they may be useful for debugging or visualization.


# Code
The paper makes use of the [ElectroMagnetic Mining Array (EMMA)](https://github.com/rpp0/emma) framework for all experiments. This framework must be installed in order to be able to train the models and use the datasets.

## Training the models
The following EMMA commands can be issued to retrain the models from scratch and perform a 10-fold cross-validation:

```bash
# Time-domain base test
emma.py 'window[0,700]' basetest ASCAD

emma.py 'window[0,700]' basetest ASCAD_desync50

emma.py 'window[0,700]' basetest ASCAD_desync100

# Time-domain CO (2 layers)
emma.py 'window[0,700]' corrtrain ASCAD --tfold

emma.py 'window[0,700]' corrtrain ASCAD_desync50 --tfold

emma.py 'window[0,700]' corrtrain ASCAD_desync100 --tfold

# Time-domain CO (1 layer)
emma.py 'window[0,700]' corrtrain ASCAD --tfold --n-hidden-layers 0

emma.py 'window[0,700]' corrtrain ASCAD_desync50 --tfold --n-hidden-layers 0

emma.py 'window[0,700]' corrtrain ASCAD_desync100 --tfold --n-hidden-layers 0

# Frequency-domain base test
emma.py 'window[0,700]' spec basetest ASCAD

emma.py 'window[0,700]' spec basetest ASCAD_desync50

emma.py 'window[0,700]' spec basetest ASCAD_desync100

# Frequency-domain CO (2 layers)
emma.py 'window[0,700]' spec corrtrain ASCAD --tfold

emma.py 'window[0,700]' spec corrtrain ASCAD_desync50 --tfold

emma.py 'window[0,700]' spec corrtrain ASCAD_desync100 --tfold

# Frequency-domain CO (1 layer)
emma.py 'window[0,700]' spec corrtrain ASCAD --tfold --n-hidden-layers 0

emma.py 'window[0,700]' spec corrtrain ASCAD_desync50 --tfold --n-hidden-layers 0

emma.py 'window[0,700]' spec corrtrain ASCAD_desync100 --tfold --n-hidden-layers 0

# Custom dataset without alignment (Figure 11)
emma.py 'rwindow[8000,10000,300]' spec 'window[0,1000]' corrtrain em-corr-arduino --valset em-cpa-arduino --batch-size 4096 --epochs 100 --max-cache 0
```

This will output model files in the `models/` directory that can be used to perform CEMA attacks and generate the figures from the paper. The paper uses the models corresponding to files ending with `-last`, which contain the state of the model after the last training epoch.

## Generating the figures
Give the trained `-last` models obtained from the above commands, the `paper_tools.py` script can automatically generate all rank figures from the paper. The paper_tools tool will automatically download models from a remote location if the they were trained on another machine.

```bash
# Individual graphs and weight plots for each model
./paper_tools.py autostats user@machine:/home/user/emma/models

# Basetest time domain
./paper_tools.py combinedtfold user@machine:/home/user/emma/models window0-700-basetest basetest

# Basetest frequency domain
./paper_tools.py combinedtfold user@machine:/home/user/emma/models window0-700-spec-basetest basetest

# Time domain CO, 2-layer MLP
./paper_tools.py combinedtfold user@machine:/home/user/emma/models window0-700-corrtrain aicorrnet-h1-leakyrelu-bn

# Frequency domain CO, 2-layer MLP
./paper_tools.py combinedtfold user@machine:/home/user/emma/models window0-700-spec-corrtrain aicorrnet-h1-leakyrelu-bn

# Time domain CO, 1-layer MLP
./paper_tools.py combinedtfold user@machine:/home/user/emma/models window0-700-corrtrain aicorrnet-h0-leakyrelu-bn

# Frequency domain CO, 1-layer MLP
./paper_tools.py combinedtfold user@machine:/home/user/emma/models window0-700-spec-corrtrain aicorrnet-h0-leakyrelu-bn
```

# How to cite

**BibTeX format**
```
@article{Robyns_Quax_Lamotte_2018,
 title={Improving CEMA using Correlation Optimization},
 volume={2019},
 url={https://tches.iacr.org/index.php/TCHES/article/view/7332},
 DOI={10.13154/tches.v2019.i1.1-24},
 number={1},
 journal={IACR Transactions on Cryptographic Hardware and Embedded Systems},
 author={Robyns, Pieter and Quax, Peter and Lamotte, Wim},
 year={2018},
 month={Nov.},
 pages={1-24}
}%      
```

**ACM format**

Robyns, P., Quax, P. and Lamotte, W. 2018. Improving CEMA using Correlation Optimization. *IACR Transactions on Cryptographic Hardware and Embedded Systems*. 2019, 1 (Nov. 2018), 1-24. DOI:https://doi.org/10.13154/tches.v2019.i1.1-24.
