# NIC College Ambassador Program --- Domain Model

Version: 1.0\
Architecture: Django + DRF Modular Monolith

## 1. Domain Overview

The NIC College Ambassador Platform manages recruitment, ambassador
lifecycle, verified profile publishing, activity tracking, events, and
administrative operations.

The system has three primary actors:

-   Visitor
-   Ambassador
-   Administrator

## 2. Core Domain Entities

## User

Purpose: Central authentication identity.

Attributes: - id - organization_email - password_hash - role -
is_active - must_change_password - created_at

Relationships: - One User belongs to one role. - Ambassador and
Administrator accounts originate from User.

------------------------------------------------------------------------

## Application

Purpose: Public ambassador recruitment submission.

Lifecycle:

submitted → under_review → approved/rejected

Attributes: - id - applicant_name - email - college - phone -
motivation - skills - status - review_notes - submitted_at

Rules: - Application approval creates User + Ambassador.

------------------------------------------------------------------------

## Ambassador

Purpose: Represents an approved NIC College Ambassador.

Attributes: - id - user_id - college - joined_date - status - level

Relationship: One Ambassador owns one published profile.

------------------------------------------------------------------------

## Ambassador Profile

Purpose: Public verified profile.

Attributes: - id - ambassador_id - bio - skills - projects -
achievements - linkedin - photo

Important Rule:

Only Verification domain can update this entity.

------------------------------------------------------------------------

## Profile Change Request

Purpose: Stores ambassador requested changes.

Lifecycle:

pending → approved/rejected

Attributes: - id - ambassador_id - section - proposed_data - status -
review_notes - created_at

------------------------------------------------------------------------

## Activity Report

Purpose: Tracks ambassador contribution and activities.

Attributes: - id - ambassador_id - title - description - evidence -
status - submitted_at

------------------------------------------------------------------------

## Event

Purpose: NIC organized events.

Attributes: - id - title - description - event_date - location - status

------------------------------------------------------------------------

## Event Registration

Purpose: Many-to-many relation between ambassadors and events.

Attributes: - id - ambassador_id - event_id - registered_at

------------------------------------------------------------------------

## Innovation Story (Future Module)

Currently removed from Phase 1.

Will be added later.

------------------------------------------------------------------------

## Audit Log

Purpose: Immutable record of administrative actions.

Attributes: - id - actor_id - action - entity_type - entity_id -
changes - timestamp

------------------------------------------------------------------------

# Domain Rules

1.  Ambassador submitted data is never directly published.
2.  Verification approval is the only path to update published profile.
3.  Every administrative action creates an audit record.
4.  Public APIs only read verified data.
5.  Admin dashboard aggregates existing domain data.
