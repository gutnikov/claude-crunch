# Knowledge System Configuration

<!-- TEMPLATE: This file is copied to .claude/knowledge/config.yaml during /init -->

```yaml
version: "1.0"
decay:
  rate: 0.01
  min_score: 0.3
  evergreen_tags:
    - architecture
    - security
    - core
    - breaking
capture:
  auto_capture_resolutions: true
  auto_capture_decisions: true
  feedback_confidence_threshold: 80
  pattern_promotion_threshold: 3
search:
  default_limit: 5
  similarity_threshold: 0.7
  boost_recent: true
```

## Configuration Options

### Decay Settings

| Setting          | Default | Description                         |
| ---------------- | ------- | ----------------------------------- |
| `rate`           | 0.01    | Daily decay rate (1% per day)       |
| `min_score`      | 0.3     | Minimum score before entry is stale |
| `evergreen_tags` | [...]   | Tags that prevent decay             |

### Capture Settings

| Setting                         | Default | Description                      |
| ------------------------------- | ------- | -------------------------------- |
| `auto_capture_resolutions`      | true    | Capture at DONE transition       |
| `auto_capture_decisions`        | true    | Capture at ENRICH (spec created) |
| `feedback_confidence_threshold` | 80      | Min review score to capture      |
| `pattern_promotion_threshold`   | 3       | Similar findings before pattern  |

### Search Settings

| Setting                | Default | Description                |
| ---------------------- | ------- | -------------------------- |
| `default_limit`        | 5       | Max results returned       |
| `similarity_threshold` | 0.7     | Min similarity for matches |
| `boost_recent`         | true    | Prefer recent entries      |
