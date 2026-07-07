# {{ project-name }}

Namespace: {{ project-name | snake_case }}
Computed: {{ top-ns }}

{% if auth %}
Authentication is included.
{% endif %}
Database:
{% case db %}
{% when sqlite %}
- SQLite
{% when postgres %}
- PostgreSQL
{% endcase %}
Bye.
