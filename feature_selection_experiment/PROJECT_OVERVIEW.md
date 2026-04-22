# Project Overview: LSTM-Autoencoder Feature Selection Comparison

## 1. Introduction
This project implements an **LSTM-Autoencoder** for anomaly detection using the **CSE-CICIDS2018** dataset. The primary goal is to evaluate and compare the impact of three different **Feature Selection** algorithms on the model's ability to detect network intrusions.

The project focuses on creating a memory-efficient pipeline capable of handling over 16 million rows of network traffic data on a machine with limited RAM, while maintaining the temporal integrity of the data required for LSTM models.

---

## 2. Dataset: CSE-CICIDS2018
The dataset consists of multiple CSV files, each representing a day of network traffic. 
- **Total Volume**: ~16 million+ rows and ~80+ columns.
- **Classes**: Benign (Normal) and various Attack types.
- **Target**: The model is trained as an Autoencoder on **Benign** data only, learning to reconstruct normal traffic. Anomalies are detected when the reconstruction error exceeds a certain threshold.

---

## 3. Feature Selection Methodology
To find the optimal subset of features, three different paradigms of feature selection are compared:

| Method | Type | Description |
| :--- | :--- | :--- |
| **Mutual Information (MI)** | Filter | Measures the statistical dependence between each feature and the target label. |
| **Random Forest Importance** | Embedded | Uses the Gini importance/mean decrease in impurity from a Random Forest model. |
| **Recursive Feature Elimination (RFE)** | Wrapper | Recursively removes the least important features using a Random Forest estimator. |

### Feature Selection Strategy for Large Data
Because loading 16M+ rows into memory for feature selection is computationally prohibitive, the project uses a **Representatively Sampled Subset**. A large, balanced sample of Benign and Attack data is drawn across all files to identify features that best distinguish normal traffic from intrusions.

---

## 4. Data Pipeline & Memory Optimization
Due to the massive size of the dataset, the project implements a **Streaming/Iterative Architecture**:

### 4.1 Memory Efficiency
- **Iterative Loading**: Data is processed file-by-file. No global concatenation of the entire dataset occurs in RAM.
- **Type Downcasting**: Numeric columns are converted from `float64` to `float32` immediately upon loading, reducing RAM usage by 50%.
- **Efficient Estimators**: `HistGradientBoostingClassifier` is used for importance calculations as it is significantly faster and more memory-efficient than standard Random Forests for millions of rows.

### 4.2 Temporal Split Logic (No Shuffle)
To preserve the time-series nature of the traffic for the LSTM, a **Sequential/Temporal Split** is applied per file:
- **Order**: Data is processed in its original sequence (no shuffling).
- **Splitting Rules for Benign Data**:
    - **If Total Benign > 1,000,000**: 
        - First 500,000 rows $\rightarrow$ **Train Set**.
        - Remaining rows $\rightarrow$ **Test Set**.
    - **If Total Benign $\le$ 1,000,000**:
        - 50:50 Split (ensuring Test $\ge$ Train).
- **Test Set Composition**: The test set consists of the Benign test portion and **all** Attack data.

---

## 5. Model Architecture
The model is a **Sequence-to-Sequence LSTM-Autoencoder**.

### 5.1 Structure
1.  **Encoder**: 
    - Input: `(batch, time_steps, num_features)`
    - LSTM layer compresses the sequence into a fixed-size hidden state (latent vector).
2.  **Latent Space**: A compressed representation of the normal traffic pattern.
3.  **Decoder**:
    - Latent vector is repeated `time_steps` times.
    - LSTM layer reconstructs the sequence from the latent representation.
    - **TimeDistributed Linear Layer**: Maps the decoder output back to the original `num_features`.

### 5.2 Hyperparameters
- **Time Steps (Window Length)**: 10
- **Hidden Dimension**: 16
- **Learning Rate**: 0.001
- **Dropout**: 0.2
- **Batch Size**: 64
- **Epochs**: 30
- **Loss Function**: Mean Absolute Error (`nn.L1Loss`)
- **Optimizer**: Adam

---

## 6. Training & Evaluation Process

### 6.1 Training Phase
- **Data**: Only $\text{Benign}_{\text{train}}$ is used.
- **Process**: The model is updated iteratively using data from each file.
- **Thresholding**: After training, the reconstruction error for the training set is calculated. The anomaly threshold is set as the **maximum reconstruction error** found in the training data:
  $$\text{Threshold} = \max(\text{Train Reconstruction Errors})$$

### 6.2 Evaluation Phase
- **Data**: $\text{Benign}_{\text{test}} + \text{All Attacks}$.
- **Detection**: If $\text{Reconstruction Error} > \text{Threshold}$, the sample is flagged as an **Anomaly**.
- **Metrics**:
    - Accuracy
    - Precision, Recall, F1-Score (Normal vs Attack)
    - ROC-AUC (Area Under the Receiver Operating Characteristic Curve)
    - PR-AUC (Area Under the Precision-Recall Curve)

---

## 7. Summary of Workflow
1.  **Sample** $\rightarrow$ **Feature Selection** (MI vs RF vs RFE) $\rightarrow$ **3 Feature Sets**.
2.  For each Feature Set:
    - **Iterative Load** $\rightarrow$ **Temporal Split** $\rightarrow$ **Train LSTM-AE on Benign**.
    - **Calculate Max Train Error** $\rightarrow$ **Set Threshold**.
    - **Iterative Load** $\rightarrow$ **Test on Benign/Attack** $\rightarrow$ **Compute Metrics**.
3.  **Compare** results to determine the best feature selection method.
