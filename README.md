# 📱 KMITL ClaimIt

<div align="center">

**An AI-powered lost and found mobile application for university campuses.**  
Connects students with lost items to university staff through image similarity matching and automated email notifications.

*Bachelor's Thesis — King Mongkut's Institute of Technology Ladkrabang, Academic Year 2024*

</div>

---

## 👥 Authors

| Name | Student ID |
|---|---|
| Kasita Sansanthad | 64011426 |
| Theint Nandar Su | 64011752 |

**Supervisor:** Asst. Prof. Dr. Pipat Sookvatana  
**Department:** Computer Engineering, School of Engineering, KMITL

---

## 📋 Table of Contents

- [Overview](#-overview)
- [Features](#-features)
- [System Architecture](#-system-architecture)
- [Tech Stack](#-tech-stack)
- [Computer Vision Pipeline](#-computer-vision-pipeline)
- [Database Schema](#-database-schema)
- [API Endpoints](#-api-endpoints)
- [Installation](#-installation)
- [Project Structure](#-project-structure)
- [Results](#-results)
- [Screenshots](#-screenshots)
- [Future Work](#-future-work)

---

## 🔍 Overview

University campuses generate hundreds of lost item reports each semester, yet most institutions rely on informal social media posts that disappear within hours. At KMITL, there was no structured digital system connecting students who lost items with the staff who found them — many items went permanently unclaimed simply because students never knew their belongings had been turned in.

**ClaimIt** solves this by:
1. Letting students report lost items with photos and descriptions
2. Letting administrators (lost and found office staff) log found items
3. Automatically comparing found items against all outstanding lost reports using **computer vision**
4. **Notifying users via email** when a visual match is found (≥70% similarity)

---

## ✨ Features

### 👤 User (Student) Side
- Register and log in with email and password
- Report lost items with photos, category, color, location, and description
- Browse all found items posted by administrators
- Search and filter found items by name, category, color, and location
- View a personalized **Recommended Items** page showing matches to your lost reports
- Receive **email notifications** when a visually similar item is found
- View and delete your own lost item reports

### 🔑 Admin (Lost & Found Office) Side
- Secure login with admin verification code
- Upload found items with photos and details
- View **dashboard statistics** — number of lost, found, and received items with a donut chart
- Browse all lost item reports with owner contact information (email)
- Search found items and lost items
- Filter lost items by category, color, and location
- Record receiver name and email when a student claims an item in person
- View all received/claimed items

---

## 🏗 System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│               Presentation Layer (Frontend)                  │
│         Flutter / Dart — UI Components & State Mgmt         │
└────────────────────────┬────────────────────────────────────┘
                         │ HTTP Requests
┌────────────────────────▼────────────────────────────────────┐
│               Application Layer (Backend)                    │
│         Django REST Framework — API Endpoints               │
│              Core Business Logic & Auth                      │
└──────────┬──────────────────────────────┬───────────────────┘
           │ ORM                          │ HTTP (CV Tasks)
┌──────────▼──────────┐     ┌────────────▼──────────────────┐
│   Data Access Layer  │     │     CV Microservice (FastAPI)  │
│   MySQL Database     │     │  Object Detection (YOLOv8m)   │
│   Views→Serializers  │     │  Background Removal (rembg)   │
│   →Models→MySQL      │     │  Image Similarity (VGG16)     │
└─────────────────────┘     └───────────────────────────────┘
```

**Three-layer architecture:**
- **Frontend** — Flutter mobile app (cross-platform, Android/iOS)
- **Backend** — Django REST API handling business logic, auth, and database operations
- **CV Microservice** — FastAPI service for all machine learning inference, isolated from the main backend

---

## 🛠 Tech Stack

### Frontend
| Technology | Version | Purpose |
|---|---|---|
| Flutter | 3.19.2 | Cross-platform mobile framework |
| Dart | 3.3.0 | Programming language |
| fl_chart | 0.68.0 | Dashboard statistics chart |
| image_picker | 1.1.2 | Camera and gallery access |
| mailer | 5.0.0 | Email notification sending |
| http | 1.2.2 | API communication |

### Backend
| Technology | Version | Purpose |
|---|---|---|
| Python | 3.9+ | Backend language |
| Django REST Framework | Latest | REST API |
| FastAPI | Latest | CV microservice |
| Uvicorn | Latest | ASGI server |
| MySQL | 8.0.39 | Relational database |

### Machine Learning / Computer Vision
| Technology | Purpose |
|---|---|
| TensorFlow / Keras | Model training and inference |
| YOLOv8m (Ultralytics) | Object detection and category prediction |
| VGG16 (pretrained, ImageNet) | Feature extraction for image similarity |
| rembg | Background removal preprocessing |
| OpenCV | Image processing |
| scikit-learn | Cosine similarity computation |
| Roboflow | Dataset labeling and management |

### Development Tools
| Tool | Purpose |
|---|---|
| Android Studio | Emulator and device testing |
| Visual Studio Code | Primary IDE |
| MySQL Workbench | Database management |
| Postman | API testing |
| Jupyter Notebook | Model training and experimentation |
| GitHub | Version control |

---

## 🤖 Computer Vision Pipeline

### 1. Object Detection — Category Prediction

When a user uploads an item photo, a YOLOv8m model predicts the item's category to assist form filling and filtering.

**Training Data**
| Class | Number of Images |
|---|---|
| Bottle | 889 |
| Calculator | 878 |
| Charger | 882 |
| Key / Car Key | 898 |
| Umbrella | 914 |
| **Total** | **~4,461** |

- Train/test split: **80% / 20%**
- Bounding box annotations created with **Roboflow**
- Data augmentation: flip, rotation, crop, brightness/contrast, noise, mosaic

**Why YOLO over image classification?**

Standard softmax classifiers assign every input to a known class — there is no output state for "this object is not in my training vocabulary." In a lost and found context, users upload arbitrary objects, so misclassifying unknown items as known categories was unacceptable.

YOLO's confidence threshold acts as a natural rejection gate: if no detection exceeds the threshold (0.70), the model returns no prediction. This is architecturally cleaner than retrofitting uncertainty estimation onto a classifier.

**Model Comparison**

| Model | Overall Accuracy | Unknown Class Rejection |
|---|---|---|
| VGG16 (fine-tuned) | 95.73% | 21.00% |
| MobileNet (fine-tuned) | 97.07% | 15.00% |
| Inception-ResNet (fine-tuned) | 98.31% | 22.00% |
| Modified VGG16 | 68.00% | 48.01% |
| YOLOv8n | 88.19% | 66.00% |
| YOLOv8s | 85.26% | 63.50% |
| **YOLOv8m ✓ Selected** | **78.52%** | **70.00%** |
| YOLOv9m | 68.84% | 73.50% |
| YOLOv10m | 57.14% | 83.00% |

**YOLOv8m** was selected as the best trade-off between known-category accuracy and unknown-class rejection.

---

### 2. Image Similarity — Match and Notify

When an administrator uploads a found item, the system compares it against all outstanding lost item reports to find potential matches.

**Pipeline:**
```
Found Item Image
      │
      ▼
Background Removal (rembg)
      │
      ▼
VGG16 Feature Extraction (ImageNet pretrained, no top layer)
      │
      ▼
Feature Vector (4096-dim)
      │
      ├── Compare with each Lost Item's feature vector
      │         (same category only)
      │
      ▼
Cosine Similarity Score
      │
      ▼
Score ≥ 0.70? ──Yes──▶ Send Email Notification to owner
      │
      No
      │
      ▼
   No action
```

**Why background removal?**

| Approach | Matching Accuracy |
|---|---|
| Raw images (no preprocessing) | 47.03% |
| Background-removed images | **80.91%** |

Background features dominated VGG16 feature vectors in raw images — two different objects photographed on the same surface scored higher similarity than the same object on two different backgrounds. Background removal preprocessing resolved this, increasing accuracy by **33.88 percentage points**.

---

## 🗄 Database Schema

**Database:** `claim_it` (MySQL 8.0.39)

### Tables

#### `app1_customuser` — Regular Users (Students)

#### `app1_itemmanager` — Administrators

#### `app1_item` — Lost and Found Items
---

## 🔌 API Endpoints

### Django REST API (Port 8000)

#### Authentication
| Method | Endpoint | Description |
|---|---|---|
| POST | `/api/register/` | Register new user |
| POST | `/api/login/` | Login, returns JWT access + refresh tokens |
| POST | `/api/token/refresh/` | Refresh access token |
| POST | `/api/admin-login/` | Admin login with verification code |

#### Items
| Method | Endpoint | Description |
|---|---|---|
| GET | `/api/found-items/` | List all found items |
| GET | `/api/lost-items/` | List all lost items |
| POST | `/api/items/` | Create new item (lost or found) |
| DELETE | `/api/items/<id>/` | Delete an item |
| PUT | `/api/items/<id>/update-received/` | Record item receiver |
| GET | `/api/received-items/` | List all received items |
| GET | `/api/my-lost-items/` | List current user's lost items |
| GET | `/api/recommended-items/` | Get similarity-matched recommendations |

#### Search and Filter
| Method | Endpoint | Description |
|---|---|---|
| GET | `/api/found-items/?search=<query>` | Search found items by name |
| GET | `/api/found-items/?category=&color=&location=` | Filter found items |
| GET | `/api/lost-items/?category=&color=&location=` | Filter lost items |

### FastAPI CV Microservice

| Method | Endpoint | Description |
|---|---|---|
| POST | `/remove-bg/` | Remove image background |
| POST | `/get-detected/` | Predict item category (YOLOv8m) |
| POST | `/get-similarity/` | Compute cosine similarity between two images |

---

## 🚀 Installation

### Prerequisites
- Flutter SDK 3.19.2+
- Python 3.9+
- MySQL 8.0+
- Android Studio (for emulator) or physical Android device

---

### 1. Clone the Repository

```bash
git clone https://github.com/tnandarsu/kmitl-claimit.git
cd kmitl-claimit
```

---

### 2. Database Setup

```sql
CREATE DATABASE claim_it;
```

Update `backend/settings.py` with your MySQL credentials:

```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'claim_it',
        'USER': 'your_mysql_user',
        'PASSWORD': 'your_mysql_password',
        'HOST': 'localhost',
        'PORT': '3306',
    }
}
```

---

### 3. Backend Setup (Django REST API)

```bash
cd backend
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate

pip install -r requirements.txt

python manage.py makemigrations
python manage.py migrate
python manage.py runserver 0.0.0.0:8000
```

---

### 4. CV Microservice Setup (FastAPI)

```bash
cd cv_service
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate

pip install -r requirements.txt

# Place your trained YOLOv8m weights in cv_service/models/
# File: best.pt

uvicorn main:app --host 0.0.0.0 --port 8001
```

---

### 5. Email Configuration

In `backend/settings.py` (or as environment variables):

```python
EMAIL_BACKEND = 'django.core.mail.backends.smtp.EmailBackend'
EMAIL_HOST = 'smtp.gmail.com'
EMAIL_PORT = 587
EMAIL_USE_TLS = True
EMAIL_HOST_USER = 'your_gmail@gmail.com'
EMAIL_HOST_PASSWORD = 'your_16_char_app_password'
```

To generate a Gmail App Password: Google Account → Security → 2-Step Verification → App Passwords.

---

### 6. Flutter App Setup

```bash
cd claimit_flutter

# Update API base URLs in lib/api/call_api.dart
# mainUrl: 'http://YOUR_SERVER_IP:8000/api'
# fastApiUrl: 'http://YOUR_SERVER_IP:8001'

flutter pub get
flutter run
```

---

## 📁 Project Structure

```
kmitl-claimit/
│
├── claimit_flutter/              # Flutter frontend
│   ├── lib/
│   │   ├── screens/              # All UI screens
│   │   │   ├── user/             # Student-facing screens
│   │   │   └── admin/            # Admin-facing screens
│   │   ├── models/               # Dart data models
│   │   ├── api/
│   │   │   └── call_api.dart     # All API calls
│   │   ├── services/
│   │   │   ├── auth_service.dart # Token management
│   │   │   └── email_sender.dart # Email notification
│   │   └── main.dart
│   └── pubspec.yaml
│
├── backend/                      # Django REST API
│   ├── app1/
│   │   ├── models.py             # Django ORM models
│   │   ├── serializers.py        # JSON serializers
│   │   ├── views.py              # API view logic
│   │   └── urls.py               # URL routing
│   ├── settings.py
│   └── requirements.txt
│
├── cv_service/                   # FastAPI CV microservice
│   ├── main.py                   # FastAPI app and endpoints
│   ├── models/
│   │   └── best.pt               # YOLOv8m trained weights
│   ├── utils/
│   │   ├── background_remover.py
│   │   ├── object_detector.py
│   │   └── similarity.py
│   └── requirements.txt
│
├── model_training/               # Jupyter notebooks
│   ├── yolo_training.ipynb
│   ├── image_similarity_testing.ipynb
│   └── classification_experiments.ipynb
│
└── README.md
```

---

## 📊 Results

| Metric | Result |
|---|---|
| Image similarity accuracy — raw images | 47.03% |
| Image similarity accuracy — after background removal | **80.91%** |
| Object detection accuracy on known categories (YOLOv8m) | 78.52% |
| Unknown-class rejection rate (YOLOv8m, threshold 0.70) | 70.00% |
| Total annotated training images | ~4,461 |
| Object classes | 5 (bottle, calculator, charger, key, umbrella) |
| Image similarity test dataset | 50 objects × 2 backgrounds |
| Email notification threshold | ≥ 0.70 cosine similarity + same category |

---

## 📸 Screenshots

### User Side

| Landing | Login | Sign Up |
|:---:|:---:|:---:|
| ![Landing](docs/screenshots/user_landing.png) | ![Login](docs/screenshots/user_login.png) | ![Signup](docs/screenshots/user_signup.png) |

| Home | Upload Form | My Lost Items |
|:---:|:---:|:---:|
| ![Home](docs/screenshots/user_home.png) | ![Upload](docs/screenshots/user_upload.png) | ![MyLost](docs/screenshots/user_my_lost.png) |

| Recommended | Found Items | Email Notify |
|:---:|:---:|:---:|
| ![Recommended](docs/screenshots/user_recommended.png) | ![Found](docs/screenshots/user_found.png) | ![Email](docs/screenshots/user_email.png) |

### Admin Side

| Dashboard | Upload Form | Lost Items |
|:---:|:---:|:---:|
| ![Dashboard](docs/screenshots/admin_dashboard.png) | ![Upload](docs/screenshots/admin_upload.png) | ![Lost](docs/screenshots/admin_lost.png) |

| Record Received | Received Items | Found Items |
|:---:|:---:|:---:|
| ![Record](docs/screenshots/admin_record.png) | ![Received](docs/screenshots/admin_received.png) | ![Found](docs/screenshots/admin_found.png) |

> Screenshots are located in `docs/screenshots/`. Place your actual screenshot files there and the table above will render correctly on GitHub.

---

## 🔮 Future Work

- **Real-time push notifications** — replace email-only alerts with in-app push notifications using Firebase Cloud Messaging
- **GPS integration** — alert students in the vicinity of a campus building when a lost item matching their report is found nearby
- **Expanded object categories** — add student cards, pencil cases, and other common lost items to the detection model with improved data collection
- **iOS support** — extend testing and deployment to iOS devices
- **Web admin portal** — browser-based interface for the lost and found office as an alternative to the mobile admin panel
- **Improved similarity model** — explore fine-tuned contrastive learning approaches (e.g. Siamese networks) for more robust image matching across varied conditions

---


## 📄 License

Copyright © 2025 School of Engineering, King Mongkut's Institute of Technology Ladkrabang.  
All rights reserved.

---

<div align="center">

Made with ❤️ at KMITL · Academic Year 2024

</div>
