# Real Estate Management System - White Label Property Management System

> **Commercial Product Disclaimer**
> This project is a generic, white-label commercial product developed under **Vectorium Technology**. The source code is private and protected by NDA. The documentation and code snippets below are provided solely for technical showcase purposes to demonstrate architectural decisions, scalability, and engineering capabilities.

---

## Project Overview

**Real Estate Management System** is a comprehensive, white-label Real Estate Management Dashboard designed to streamline the entire property lifecycle for agencies of any size. From portfolio management to automated lead conversion, it serves as a central hub for agents to manage properties, track client interactions, and automate sales workflows.

Designed with a **Multi-Tenant Architecture** mindset, the system features a robust **Business Rules Engine** that orchestrates lead assignments and enforces stage transitions, adaptable to different agency workflows.

## Tech Stack

The application utilizes a hybrid architecture combining a secure, data-heavy backend with a dynamic, responsive frontend.

### Backend & Core
* **Python / Django 5.x**: Core application framework implementing the Modular Monolith pattern.
* **Django Rest Framework (DRF)**: RESTful API layer for frontend data consumption.
* **PostgreSQL**: Relational database optimized for complex property-client relationships.
* **Celery / Redis**: Asynchronous task processing for notifications and heavy report generation.

### Frontend & Interface
* **React / Next.js Integration**: Dynamic user interfaces for dashboards and forms.
* **SWR (Stale-While-Revalidate)**: Advanced data fetching, caching, and optimistic UI updates.
* **Axios**: Http client with centralized interceptors for JWT Authentication.
* **Bootstrap 5 / SCSS**: Modular design system based on Argon Dashboard principles.

### Integrations & DevOps
* **NetGSM / Twilio**: Integrated SMS gateway for automated client notifications.
* **WhatsApp Business API**: Direct message integration for lead engagement.
* **Docker**: Containerized development and deployment environment.

---

## Key Features

### 1. Advanced Property Lifecycle Management
Manage the complete lifecycle of a property listing with detailed validation and status tracking.
* **Dynamic Attributes**: Support for residential, commercial, and land property types with specific fields.
* **Image Gallery**: Drag-and-drop upload with server-side optimization (Pillow) and storage.
* **Status Workflow**: Active, Deposit Taken, Sold, Withdrawn statuses with full audit logging.

### 2. Intelligent Lead Management (CRM)
A sophisticated Sales Process module that guides agents through a standardized sales flow.
* **Stage State Machine**: Enforces valid transitions (e.g., *Lead* -> *Needs Analysis* -> *Offer Sent*).
* **Automated Task Generation**: Automatically creates follow-up tasks (Calls, Meetings) when a lead enters a new stage.
* **Activity Scoring**: Calculates "Lead Heat" based on interaction frequency and stage duration.

### 3. Algorithmic Workload Balancing
Algorithmically distributes incoming leads and tasks among agents to ensure optimal performance.
* **Score-Based Assignment**: Evaluates current active leads, overdue tasks, and success rates to assign new potential clients.
* **Performance Metrics**: Real-time dashboards showing agent conversion rates and response times.

---

## Engineering Highlights

### Business Rules Engine
The core of the CRM is handled by a centralized `SalesProcessRules` class that encapsulates all business logic, decoupling it from Views and Models. This ensures consistent validation across API endpoints and admin actions.

```python
# apps/sales_process/business_rules.py

class SalesProcessRules:
    """
    Centralized business logic for Sales Process management.
    Handles stage transitions, automation validation, and workload scoring.
    """
    
    @staticmethod
    def validate_stage_transition(lead, from_stage, to_stage, user):
        """
        Validates if a lead can move from one stage to another based on strict rules.
        """
        errors = []
        
        # 1. Check State Machine Flow
        if not SalesProcessRules.can_move_to_stage(lead, to_stage):
            errors.append(f"Invalid transition from {from_stage.name} to {to_stage.name}.")
        
        # 2. Check Prerequisites (e.g., Mandatory WhatsApp Offer)
        if to_stage.name == 'offer_sent':
            whatsapp_messages = WhatsAppMessage.objects.filter(
                lead=lead, message_type='offer_sent', status='sent'
            )
            if not whatsapp_messages.exists():
                errors.append("Offer must be sent via WhatsApp before moving to this stage.")
        
        # 3. Check Data Completeness
        elif to_stage.name == 'contract_signed':
            if not lead.phone or not lead.email:
                errors.append("Contact details are required for Contract stage.")

        return {'is_valid': len(errors) == 0, 'errors': errors}

    @staticmethod
    def get_workload_balance():
        """
        Calculates agent workload score to optimize lead distribution.
        Score = (Active Leads * 2) + (Pending Tasks * 1) + (Overdue Tasks * 3)
        """
        staff_workload = User.objects.filter(is_active=True, is_staff=True).annotate(
            active_leads=Count('assigned_leads', filter=Q(assigned_leads__current_stage__name__in=['active_'])),
            overdue_tasks=Count('assigned_tasks', filter=Q(status='overdue'))
        )
        # Logic to return sorted list of agents by availability...

```

### Client-Side Data Layer & Caching

To ensure a snappy user experience for data-heavy dashboards, we implemented a Service Layer using **Axios** and **SWR**. This provides intelligent caching, automatic revalidation, and request deduplication.

```javascript
// apps/api/PropertyService.js

import useSWR from 'swr';
import axios from 'axios';

class PropertyService {
  constructor() {
    this.axios = axios.create({ baseURL: API_BASE_URL });
    
    // Automatic Token Refresh Interceptor
    this.axios.interceptors.response.use(
      (response) => response,
      async (error) => {
        if (error.response?.status === 401) {
          await this.refreshToken();
          return this.axios.request(error.config); // Retry request
        }
        return Promise.reject(error);
      }
    );
  }

  // React Hook for consuming Property Data with SWR Caching
  useAllProperties = (params = {}) => {
    const queryString = new URLSearchParams(params).toString();
    
    const { data, error, mutate } = useSWR(
      `${API_BASE_URL}/api/properties/?${queryString}`,
      this.fetcher,
      {
        dedupingInterval: 300000, // Cache for 5 minutes
        revalidateOnFocus: false
      }
    );

    return {
      properties: data?.results || [],
      count: data?.count || 0,
      isLoading: !error && !data,
      isError: !!error,
      mutate
    };
  }
}

```

---

## Dashboard Architecture

The frontend interfaces are built to handle high data density with ease.

1. **Kanban View**: For visual Lead Management, moving cards triggers API calls to `SalesProcessRules.validate_stage_transition`.
2. **Advanced Data Tables**: Server-side pagination, sorting, and filtering (by price range, location, features) directly integrated with Django `FilterSets`.
3. **Role-Based Access**: Views differ significantly for Admin users versus Standard Agents, controlled via Backend Permissions and Frontend Route Guards.

---

## Visual Gallery

| Dashboard Overview | Property Management Form | Kanban Sales Pipeline |
| --- | --- | --- |
|  |  |  |
| *High-level metrics on Sales and Properties* | *Multi-step Form with validation* | *Drag-and-drop Lead Management* |

---

**Developed by the Engineering Team at Vectorium Technology**
