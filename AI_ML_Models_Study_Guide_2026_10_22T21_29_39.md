


---

## 1. Traditional ML Models & Algorithms

### Supervised Learning (Labeled Data → Predictions)
- **Linear Regression** — Predicts continuous numerical values (e.g., house prices)
- **Logistic Regression** — Binary classification (e.g., spam vs. not spam)
- **Decision Trees** — Tree-based splits for classification/regression
- **Random Forests** — Ensemble of decision trees for improved accuracy
- **XGBoost / Gradient Boosting** — High-performance ensemble method; popular in competitions
- **Support Vector Machines (SVM)** — Finds optimal hyperplane for classification
- **k-Nearest Neighbors (k-NN)** — Classifies based on proximity to labeled neighbors

### Unsupervised Learning (Unlabeled Data → Patterns)
- **K-Means Clustering** — Groups data into k clusters based on similarity
- **Principal Component Analysis (PCA)** — Dimensionality reduction
- **Anomaly Detection** — Identifies outliers (e.g., fraud detection)
- **Association Rules** — Finds relationships (e.g., market basket analysis)

### Reinforcement Learning (Agent → Reward Optimization)
- Agent interacts with an environment, receives rewards/penalties
- Learns optimal policy through trial and error
- Key in RLHF (Reinforcement Learning from Human Feedback) for fine-tuning LLMs

### Self-Supervised Learning
- Model generates its own labels from the data (e.g., masked language modeling in BERT, next-token prediction in GPT)
- Foundation for most modern LLM pre-training

---

## 2. Deep Learning / Neural Network Architectures

### Artificial Neural Networks (ANNs)
- Basic feedforward networks with input, hidden, and output layers
- Use activation functions (ReLU, sigmoid, softmax)
- Trained via backpropagation and gradient descent

### Convolutional Neural Networks (CNNs)
- Specialized for **image and video** tasks
- Use convolutional filters to detect spatial features (edges, textures, objects)
- Applications: image classification, object detection, computer vision

### Recurrent Neural Networks (RNNs)
- Designed for **sequential data** (time-series, text)
- Maintain hidden state across time steps
- **LSTM (Long Short-Term Memory)** — Solves vanishing gradient problem; better at long sequences
- **GRU (Gated Recurrent Unit)** — Simplified LSTM variant

### Transformers ⭐ (Most Important for This Exam)
- **Self-Attention Mechanism** — Allows the model to weigh the importance of all tokens in a sequence simultaneously
- **Encoder-Decoder Architecture** — Encoder processes input; decoder generates output
- **Encoder-only** (e.g., BERT) — Good for classification, embeddings
- **Decoder-only** (e.g., GPT, Claude, Llama) — Good for text generation
- **Encoder-Decoder** (e.g., T5, BART) — Good for translation, summarization
- Processes tokens in **parallel** (unlike RNNs), enabling massive scale

---

## 3. Generative AI Model Types

### Large Language Models (LLMs)
- Based on Transformer architecture (typically decoder-only)
- Trained on massive text corpora using next-token prediction
- Capabilities: text generation, summarization, translation, Q&A, coding, reasoning
- Examples: GPT-4, Claude 3, Llama 3, Mistral, Amazon Titan Text

### Diffusion Models
- Generate images by starting from random noise and iteratively **denoising**
- Forward process: adds noise to data; Reverse process: learns to remove noise
- Examples: Stable Diffusion, Amazon Titan Image Generator, DALL-E
- Applications: image generation, image editing, inpainting

### Generative Adversarial Networks (GANs)
- Two competing neural networks:
  - **Generator** — Creates fake data
  - **Discriminator** — Tries to distinguish real from fake
- Trained adversarially until generator produces realistic outputs
- Applications: image synthesis, style transfer, data augmentation

### Variational Autoencoders (VAEs)
- **Encoder** compresses input into a latent space distribution
- **Decoder** reconstructs/generates new data from latent space
- Applications: image generation, anomaly detection

### Multimodal Models
- Process and generate across multiple modalities (text, image, audio, video)
- Examples: GPT-4V, Claude 3 (vision), Amazon Nova, Gemini

---

## 4. Foundation Models on Amazon Bedrock

### What is a Foundation Model?
A large model **pre-trained on broad data at scale** that can be adapted to a wide range of downstream tasks through prompting, fine-tuning, or tool use.

### Models Available on Bedrock

| Model Family | Provider | Type | Key Strengths |
|---|---|---|---|
| **Claude 3** (Opus, Sonnet, Haiku) | Anthropic | Text + Vision | Reasoning, analysis, coding, long context (200K tokens) |
| **Llama 3** | Meta | Text | Open-weight, general-purpose, multilingual |
| **Titan Text** | Amazon | Text | General text generation, summarization |
| **Titan Embeddings** | Amazon | Embeddings | Text/multimodal embeddings for RAG |
| **Titan Image Generator** | Amazon | Image | Text-to-image, image editing |
| **Nova** | Amazon | Multimodal | Understanding and generation across modalities |
| **Jurassic-2** | AI21 Labs | Text | Summarization, paraphrasing |
| **Mistral / Mixtral** | Mistral AI | Text | Efficient, multilingual, coding |
| **Command R / R+** | Cohere | Text | Enterprise search, RAG-optimized |
| **Embed** | Cohere | Embeddings | Multilingual embeddings |
| **Stable Diffusion XL** | Stability AI | Image | High-quality image generation |

### When to Use Bedrock vs. SageMaker
- **Bedrock** → You want managed access to foundation models without managing infrastructure; prompt engineering, RAG, agents
- **SageMaker** → You need full control over training, custom models, MLOps, explainability

---

## 5. Key Generative AI Concepts

### Tokenization
- Breaking text into smaller units (tokens) — words, subwords, or characters
- Models process tokens, not raw text
- Token count affects cost and context window limits

### Embeddings
- Dense vector representations of tokens/text/images
- Capture semantic meaning; similar concepts have similar vectors
- Used for search, RAG, clustering, classification

### Vector Databases
- Store and index embeddings for fast similarity search
- Used in RAG pipelines to retrieve relevant context
- Examples: Amazon OpenSearch Serverless, Pinecone, FAISS

### Prompt Engineering Techniques
- **Zero-shot** — No examples provided; rely on model's training
- **One-shot** — One example provided
- **Few-shot** — Multiple examples provided
- **Chain-of-Thought (CoT)** — Ask the model to reason step-by-step
- **ReAct** — Combines reasoning and acting (tool use)
- **Tree-of-Thought** — Explores multiple reasoning paths

### Inference Parameters
- **Temperature** — Controls randomness (0 = deterministic, 1 = creative)
- **Top-p (nucleus sampling)** — Limits token selection to cumulative probability p
- **Top-k** — Limits selection to top k most probable tokens
- **Max tokens** — Maximum length of generated output
- **Stop sequences** — Tokens that signal the model to stop generating

### Fine-Tuning Techniques
- **SFT (Supervised Fine-Tuning)** — Train on labeled instruction-response pairs
- **RLHF (Reinforcement Learning from Human Feedback)** — Human preferences guide model alignment
- **DPO (Direct Preference Optimization)** — Simplified alternative to RLHF
- **LoRA (Low-Rank Adaptation)** — Efficient fine-tuning by updating small adapter layers
- **PEFT (Parameter-Efficient Fine-Tuning)** — Umbrella term for methods like LoRA

### RAG (Retrieval-Augmented Generation)
- Combines a retrieval system with an LLM
- Steps: Query → Retrieve relevant documents → Augment prompt with context → Generate response
- Reduces hallucinations and provides up-to-date information
- AWS Implementation: **Bedrock Knowledge Bases** (connects to S3, OpenSearch, etc.)

---

## 6. Model Evaluation Metrics

### Classification Metrics
- **Accuracy** — Overall correct predictions / total predictions
- **Precision** — True positives / (True positives + False positives)
- **Recall (Sensitivity)** — True positives / (True positives + False negatives)
- **F1 Score** — Harmonic mean of precision and recall
- **AUC-ROC** — Area under the Receiver Operating Characteristic curve

### Regression Metrics
- **MSE (Mean Squared Error)** — Average of squared differences
- **RMSE (Root Mean Squared Error)** — Square root of MSE
- **MAE (Mean Absolute Error)** — Average of absolute differences

### GenAI-Specific Metrics
- **BLEU** — Measures n-gram overlap (machine translation)
- **ROUGE** — Measures recall of n-grams (summarization)
- **BERTScore** — Semantic similarity using embeddings
- **Perplexity** — How well the model predicts the next token (lower = better)
- **Human Evaluation** — Fluency, relevance, helpfulness, harmlessness
- **Toxicity / Bias Scores** — Measure harmful or biased outputs

### Model Fit Concepts
- **Underfitting** — Model too simple; high bias, poor training performance
- **Overfitting** — Model too complex; low training error but poor generalization
- **Bias-Variance Tradeoff** — Balance between underfitting and overfitting

---

## 7. AWS AI/ML Services Quick Reference

| Service | Category | What It Does |
|---|---|---|
| **Amazon Bedrock** | GenAI | Managed access to foundation models, agents, guardrails, knowledge bases |
| **Amazon SageMaker** | ML Platform | Full ML lifecycle — build, train, deploy, monitor |
| **SageMaker JumpStart** | ML Platform | Pre-trained model hub for quick deployment |
| **SageMaker Canvas** | ML Platform | No-code ML predictions for business users |
| **SageMaker Clarify** | Responsible AI | Bias detection, feature importance, model explainability (SHAP) |
| **SageMaker Model Monitor** | MLOps | Detect data/model drift in production |
| **SageMaker Ground Truth** | Data Labeling | Managed data labeling with human + ML |
| **SageMaker Data Wrangler** | Data Prep | Visual data preparation and transformation |
| **SageMaker Feature Store** | Data | Centralized feature repository |
| **Amazon Q** | AI Assistant | Business Q&A, developer assistance, QuickSight integration |
| **Amazon Rekognition** | Computer Vision | Image/video analysis, face detection, content moderation |
| **Amazon Comprehend** | NLP | Sentiment analysis, entity recognition, language detection |
| **Amazon Textract** | Document AI | Extract text, tables, forms from documents |
| **Amazon Transcribe** | Speech | Speech-to-text transcription |
| **Amazon Polly** | Speech | Text-to-speech synthesis |
| **Amazon Translate** | NLP | Real-time language translation |
| **Amazon Lex** | Conversational AI | Build chatbots and voice bots |
| **Amazon Kendra** | Search | Intelligent enterprise search |
| **Amazon Personalize** | ML | Real-time personalization and recommendations |
| **Amazon Forecast** | ML | Time-series forecasting |
| **Amazon Fraud Detector** | ML | Identify potentially fraudulent activities |
| **Amazon CodeWhisperer** | Developer | AI-powered code suggestions |
| **AWS HealthScribe** | Healthcare | Clinical documentation AI |

---

## 8. Responsible AI Concepts

- **Bias Types** — Selection bias, measurement bias, confirmation bias, automation bias, societal bias
- **Fairness** — Equal treatment across demographic groups
- **Explainability** — SHAP (SHapley Additive exPlanations), LIME (Local Interpretable Model-agnostic Explanations)
- **Guardrails for Bedrock** — Content filtering, denied topics, PII redaction, word filters
- **Model Cards** — Documentation of model purpose, performance, limitations
- **Frameworks** — NIST AI RMF, EU AI Act, ISO/IEC 42001

---

## 9. Security & Governance

- **IAM Policies** — Control access to AI services
- **Encryption** — KMS for data at rest, TLS for data in transit
- **VPC Endpoints** — Private connectivity to AI services
- **Amazon Macie** — Discover and protect sensitive data
- **AWS CloudTrail** — Audit API calls to AI services
- **Data Residency** — GDPR, HIPAA compliance considerations
- **MLOps** — Model versioning, CI/CD for ML, drift detection

---

## Quick Decision Framework: Which Service to Use?

| Scenario | Service |
|---|---|
| Access pre-built foundation models via API | **Amazon Bedrock** |
| Build and train custom ML models | **Amazon SageMaker** |
| No-code ML for business analysts | **SageMaker Canvas** |
| Deploy open-source models quickly | **SageMaker JumpStart** |
| Build a RAG application | **Bedrock Knowledge Bases** |
| Create an AI agent with tool use | **Bedrock Agents** |
| Detect bias in training data/models | **SageMaker Clarify** |
| Monitor model performance in production | **SageMaker Model Monitor** |
| Add content safety filters to GenAI apps | **Guardrails for Bedrock** |
| Extract text from scanned documents | **Amazon Textract** |
| Analyze sentiment in customer reviews | **Amazon Comprehend** |
| Build a conversational chatbot | **Amazon Lex** |
| Generate personalized recommendations | **Amazon Personalize** |

---

> **Exam Tip:** The heaviest domain (28%) is *Applications of Foundation Models*. Focus on Amazon Bedrock, prompt engineering, RAG, fine-tuning, and knowing when to use Bedrock vs. SageMaker.

> **Exam Tip:** Understand the difference between model types — when to use a Transformer-based LLM vs. a Diffusion model vs. a traditional ML algorithm.

> **Exam Tip:** Responsible AI (14%) is a guaranteed scoring area — know bias types, SHAP/LIME, Guardrails for Bedrock, and SageMaker Clarify.
