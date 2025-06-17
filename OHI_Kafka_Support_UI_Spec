# Technical Implementation Specification for OHI Kafka Support

Based on a comprehensive analysis of the Message Queues codebase, this document provides a detailed technical specification for adding OHI (On Host Integration) Kafka support to the entire project.

## Table of Contents
1. [Entity Type Definitions](#1-entity-type-definitions)
2. [Constants and Configuration Updates](#2-constants-and-configuration-updates)
3. [Query Utils Updates](#3-query-utils-updates)
4. [Data Utils Updates](#4-data-utils-updates)
5. [Metric Fetching Hook Updates](#5-metric-fetching-hook-updates)
6. [UI Component Updates](#6-ui-component-updates)
7. [Test Coverage Updates](#7-test-coverage-updates)
8. [E2E Test Updates](#8-e2e-test-updates)
9. [Implementation Checklist](#9-implementation-checklist)
10. [Migration Considerations](#10-migration-considerations)

## 1. Entity Type Definitions

### New Entity Types Required
Following the existing pattern `[PROVIDER][RESOURCE_TYPE]`, the OHI Kafka entities should be:
- **Cluster**: `ONHOSTKAFKACLUSTER`
- **Topic**: `ONHOSTKAFKATOPIC`
- **Broker**: `ONHOSTKAFKABROKER` (if applicable)

### Domain Configuration
All entities should belong to the `INFRA` domain, consistent with existing Kafka integrations.

## 2. Constants and Configuration Updates

### File: `/common/config/constants.ts`

```typescript
// Add to PROVIDERS_ID_MAP
export const PROVIDERS_ID_MAP: any = {
  'AWS MSK': 'aws',
  'Confluent Cloud': 'confluent_cloud',
  'OHI Kafka': 'ohi_kafka',  // NEW
};

// Add new provider constant
export const OHI_KAFKA_PROVIDER = 'OHI Kafka';  // NEW

// Update KAFKA_PROVIDERS array
export const KAFKA_PROVIDERS = [MSK_PROVIDER, CONFLUENT_CLOUD_PROVIDER, OHI_KAFKA_PROVIDER];

// Update domain filters
export const ALL_CLUSTER_DOMAIN = `domain IN ('INFRA') AND type IN ('AWSMSKCLUSTER', 'CONFLUENTCLOUDCLUSTER', 'ONHOSTKAFKACLUSTER')`;
export const ALL_TOPIC_DOMAIN = `domain IN ('INFRA') AND type IN ('AWSMSKTOPIC', 'CONFLUENTCLOUDKAFKATOPIC', 'ONHOSTKAFKATOPIC')`;

// Add OHI hidden filters
export const OHI_HIDDEN_FILTERS_CONDITION = "metricName like '%kafka%' AND instrumentation.provider = 'kafka'";

// Update HIDDEN_FILTERS
export const HIDDEN_FILTERS = "metricName like '%aws.kafka%' or metricName like '%confluent%' or provider IN ('AwsMskCluster', 'AwsMskBroker', 'AwsMskTopic') or (metricName like '%kafka%' AND instrumentation.provider = 'kafka')";

// Add OHI event types (if any specific samples exist)
export const OHI_KAFKA_EVENT_TYPES = ['Metric'];  // OHI typically uses Metric events

// Add cluster column mapping for OHI
export const CLUSTER_COLUMNS_MAPPING = (metricStream: boolean, provider: string) => {
  if (provider === MSK_PROVIDER) {
    return !metricStream ? 'provider.clusterName' : '`aws.kafka.ClusterName` OR `aws.msk.clusterName`';
  } else if (provider === CONFLUENT_CLOUD_PROVIDER) {
    return '`kafka.cluster_name` or `kafka.clusterName` or `confluent.clusterName`';
  } else if (provider === OHI_KAFKA_PROVIDER) {
    return '`kafka.cluster.name` or `clusterName`';  // Based on typical OHI attribute patterns
  }
  return '';
};
```

## 3. Query Utils Updates

### File: `/common/utils/query-utils.ts`

```typescript
// Add OHI entity query filters
export const OHI_CLUSTER_QUERY_FILTER = "domain IN ('INFRA') AND type='ONHOSTKAFKACLUSTER'";
export const OHI_TOPIC_QUERY_FILTER = "domain IN ('INFRA') AND type='ONHOSTKAFKATOPIC'";
export const OHI_BROKER_QUERY_FILTER = "domain IN ('INFRA') AND type='ONHOSTKAFKABROKER'";

// Add OHI query filter functions
export const OHI_CLUSTER_QUERY_FILTER_FUNC = (searchName: string) => {
  return `${OHI_CLUSTER_QUERY_FILTER} ${searchName ? `AND name IN (${searchName})` : ''}`;
};

export const OHI_TOPIC_QUERY_FILTER_FUNC = (searchName: string) => {
  return `${OHI_TOPIC_QUERY_FILTER} AND name IN (${searchName})`;
};

// Update COUNT_TOPIC_QUERY_FILTER
export const COUNT_TOPIC_QUERY_FILTER = `domain IN ('INFRA') AND type IN ('AWSMSKTOPIC', 'CONFLUENTCLOUDKAFKATOPIC', 'ONHOSTKAFKATOPIC')`;

// Add OHI queries object
export const OHI_KAFKA_QUERIES: QueryOptions = {
  CLUSTER_INCOMING_THROUGHPUT: {
    select: `sum(bytesInPerSec)`,
    from: {
      select: `average(kafka.net.bytes_in.rate) as 'bytesInPerSec'`,
      from: 'Metric',
      where: ["instrumentation.provider = 'kafka'"],
      facet: ['kafka.cluster.name or clusterName as cluster', 'kafka.broker.id'],
      limit: MAX_LIMIT,
    },
    isNested: true,
  },
  CLUSTER_OUTGOING_THROUGHPUT: {
    select: `sum(bytesOutPerSec)`,
    from: {
      select: `average(kafka.net.bytes_out.rate) as 'bytesOutPerSec'`,
      from: 'Metric',
      where: ["instrumentation.provider = 'kafka'"],
      facet: ['kafka.cluster.name or clusterName as cluster', 'kafka.broker.id'],
      limit: MAX_LIMIT,
    },
    isNested: true,
  },
  CLUSTER_HEALTH_QUERY: {
    select: `latest(activeControllerCount) as 'Active Controllers', latest(offlinePartitionsCount) as 'Offline Partitions'`,
    from: 'Metric',
    where: ["instrumentation.provider = 'kafka'", "metricName IN ('kafka.controller.active.count', 'kafka.partition.offline')"],
    facet: 'kafka.cluster.name or clusterName as cluster',
    limit: MAX_LIMIT,
  },
  TOPIC_HEALTH_QUERY: {
    select: `latest(\`Bytes In\`) as 'Bytes In', latest(\`Bytes Out\`) as 'Bytes Out'`,
    from: {
      select: `average(kafka.topic.bytes_in.rate) as 'Bytes In', average(kafka.topic.bytes_out.rate) as 'Bytes Out'`,
      from: 'Metric',
      where: ["instrumentation.provider = 'kafka'"],
      facet: ["kafka.topic as 'topic'"],
      limit: MAX_LIMIT,
    },
    facet: 'topic',
    limit: MAX_LIMIT,
    isNested: true,
  },
  // Add more query definitions as needed
};

// Update TOTAL_CLUSTERS_QUERY function
export const TOTAL_CLUSTERS_QUERY = (provider: string, isMetricStream: boolean) => {
  // ... existing code ...
  else if (provider === OHI_KAFKA_PROVIDER) {
    return {
      select: `uniqueCount(kafka.cluster.name or clusterName) as 'Total clusters'`,
      from: 'Metric',
      where: ["instrumentation.provider = 'kafka'"],
      metricType: 'Cluster',
    };
  }
  return {};
};

// Update ALL_KAFKA_TABLE_QUERY to include OHI
export const ALL_KAFKA_TABLE_QUERY = ngql`query ALL_KAFKA_TABLE_QUERY($awsQuery: String!, $confluentCloudQuery: String!, $ohiQuery: String!, $facet: EntitySearchCountsFacet!,$orderBy: EntitySearchOrderBy!) {
  actor {
    awsEntitySearch: entitySearch(query: $awsQuery) {
      // ... existing fields ...
    }
    confluentCloudEntitySearch: entitySearch(query: $confluentCloudQuery) {
      // ... existing fields ...
    }
    ohiEntitySearch: entitySearch(query: $ohiQuery) {
      count
      facetedCounts(facets: {facetCriterion: {facet: $facet}, orderBy: $orderBy}) {
        counts {
          count
          facet
        }
      }
      results {
        accounts {
          id
          name
        }
      }
    }
  }
}`;

// Add OHI topics table query
export const getTopicsTableQuery: any = (queryKey: string, orderBy: string, where: any[], attributeSortMapping: { [key: string]: string }) => {
  const queries: { [key: string]: any } = {
    // ... existing queries ...
    ohi_kafka: {
      select: ['guid', 'name', 'Topic', 'bytesInPerSec', 'bytesOutPerSec', 'messagesInPerSec'],
      from: {
        select: [
          "average(kafka.topic.bytes_in.rate) or 0 AS 'bytesInPerSec'",
          "average(kafka.topic.bytes_out.rate) or 0 AS 'bytesOutPerSec'",
          "average(kafka.topic.messages_in.rate) or 0 AS 'messagesInPerSec'",
        ],
        from: 'Metric',
        facet: [
          "entity.guid as 'guid'",
          "entity.name OR entityName as 'name'",
          "kafka.topic AS 'Topic'",
        ],
        orderBy: `average(${attributeSortMapping[orderBy]} or 0)`,
        limit: MAX_LIMIT,
        metricType: 'Topic',
      },
      where: ["instrumentation.provider = 'kafka' AND kafka.topic IS NOT NULL", ...where],
      orderBy: `${orderBy} DESC`,
      limit: LIMIT_20,
      isNested: true,
    },
  };
  return queries[queryKey];
};

// Add OHI summary queries
const SUMMARY_QUERIES: any = (staticInfo: StaticInfo) => {
  return {
    // ... existing queries ...
    ohi_kafka: {
      [METRIC_IDS.TOTAL_CLUSTERS]: {
        from: 'Metric',
        select: `uniqueCount(kafka.cluster.name or clusterName) as 'Total clusters'`,
        where: ["instrumentation.provider = 'kafka'"],
      },
      [METRIC_IDS.UNHEALTHY_CLUSTERS]: {
        from: 'Metric',
        select: `${staticInfo.unhealthyClusters} as 'Unhealthy clusters'`,
        where: ["instrumentation.provider = 'kafka'"],
      },
      [METRIC_IDS.BROKERS]: {
        from: 'Metric',
        select: `uniqueCount(kafka.broker.id) as 'Brokers'`,
        where: ["instrumentation.provider = 'kafka'"],
        metricType: 'Broker',
      },
      [METRIC_IDS.TOPICS]: {
        from: 'Metric',
        select: `uniqueCount(kafka.topic) as 'Topics'`,
        where: ["instrumentation.provider = 'kafka'"],
      },
      // Add more metric definitions as needed
    },
  };
};
```

## 4. Data Utils Updates

### File: `/common/utils/data-utils.ts`

```typescript
// Update prepareTableData to handle OHI entities
export function prepareTableData(data: any): {
  headers: string[];
  items: MessageQueueMetaRowItem[];
} {
  // ... existing code ...
  
  const ohiKafkaItems = data.actor.ohiEntitySearch?.results?.accounts?.map(
    (acc: any, index: number) => {
      return {
        'Name': [acc['name'], `${MESSAGE_QUEUE_TYPE} - ${OHI_KAFKA_PROVIDER}`],
        'Clusters': data.actor.ohiEntitySearch.facetedCounts.counts[index].count,
        'Provider': OHI_KAFKA_PROVIDER,
        'Account Name': acc['name'],
        'Total/Unhealthy Clusters': data.actor.ohiEntitySearch.facetedCounts.counts[index].count,
        'Health': [0, 0],
        'Incoming Throughput': 0,
        'Outgoing Throughput': 0,
        'Account Id': acc['id'],
        'Is Metric Stream': false,  // OHI doesn't use metric streams
        'hasError': false,
      };
    },
  ) || [];

  const combinedItems = [...awsItems, ...confluentCloudItems, ...ohiKafkaItems];
  combinedItems.sort((a, b) => a['Account Id'] - b['Account Id']);
  return { headers, items: combinedItems };
}

// Update isUnhealthyEntity to handle OHI metrics
export const isUnhealthyEntity = (m: Metric, provider: string, show: string): boolean => {
  if (provider === CONFLUENT_CLOUD_PROVIDER) {
    // ... existing code ...
  } else if (provider === OHI_KAFKA_PROVIDER) {
    if (show === 'topic') {
      return (m.name === 'Bytes In' && m.value === 0) || 
             (m.name === 'Bytes Out' && m.value === 0);
    }
    return (
      (m.name === 'Active Controllers' && m.value !== 1) ||
      (m.name === 'Offline Partitions' && m.value > 0) ||
      (m.name === 'Under Replicated Partitions' && m.value > 0)
    );
  }
  // ... existing code ...
};

// Update getEntityHealth for OHI
export const getEntityHealth = (EntityMetrics: EntityMetrics, provider: string, show: string): ALERT_SEVERITY => {
  // ... existing code ...
  if (provider === OHI_KAFKA_PROVIDER && show === 'cluster') {
    // Add OHI-specific health logic if needed
  }
  // ... existing code ...
};

// Update getPredefinedFiltersForSummary for OHI
export const getPredefinedFiltersForSummary = (
  accountName: string,
  provider: string,
  isMetricStream: boolean,
  extraCluster: string,
  extraTopic: string,
  extraBrokerId: string,
): Filter[] => {
  // ... existing code ...
  
  if (provider === OHI_KAFKA_PROVIDER) {
    filterData.push({
      id: filterLabels[2],
      term: {
        label: filterLabels[2],
        value: 'kafka.cluster.name or clusterName',
      },
      operator: { label: '=', multiple: true, value: '=' },
      value: Clusters,
      writeable: true,
    });
    
    filterData.push({
      id: 'Broker Ids',
      term: {
        label: 'Broker Ids',
        value: 'kafka.broker.id',
      },
      operator: { label: '=', multiple: true, value: '=' },
      value: Brokers,
      writeable: true,
    });
    
    filterData.push({
      id: 'Topics',
      term: {
        label: 'Topics',
        value: 'kafka.topic',
      },
      operator: { label: '=', multiple: true, value: '=' },
      value: Topics,
      writeable: true,
    });
  }
  
  return filterData;
};

// Update getClusterFilterSet for OHI
export const getClusterFilterSet = (clusters: string, provider: string, isMetricStream: boolean) => {
  if (!clusters) return [];
  // ... existing code ...
  else if (provider === OHI_KAFKA_PROVIDER) {
    return [`(\`kafka.cluster.name\` or \`clusterName\`) IN (${clusters})`];
  }
  // ... existing code ...
};
```

## 5. Metric Fetching Hook Updates

### File: `/common/hooks/use-fetch-entity-metrics/index.ts`

```typescript
export function useFetchEntityMetrics({
  item,
  filterSet,
  show = 'cluster',
  groupBy,
  updateItems,
}: {
  // ... existing parameters ...
}): {
  // ... existing return type ...
} {
  // ... existing code ...
  
  let baseHealthQuery: any = isMetricStream
    ? MTS_QUERIES[`${show.toUpperCase()}_HEALTH_QUERY`]
    : item.Provider === CONFLUENT_CLOUD_PROVIDER
      ? CONFLUENT_CLOUD_DIM_QUERIES[`${show.toUpperCase()}_HEALTH_QUERY`]
      : item.Provider === OHI_KAFKA_PROVIDER
        ? OHI_KAFKA_QUERIES[`${show.toUpperCase()}_HEALTH_QUERY`]  // NEW
        : DIM_QUERIES[`${show.toUpperCase()}_HEALTH_QUERY`];

  baseHealthQuery = groupBy
    ? isMetricStream
      ? MTS_QUERIES[`${show.toUpperCase()}_GROUPBY_${groupBy.toUpperCase()}`]
      : item.Provider === CONFLUENT_CLOUD_PROVIDER
        ? CONFLUENT_CLOUD_DIM_QUERIES[`${show.toUpperCase()}_GROUPBY_${groupBy.toUpperCase()}`]
        : item.Provider === OHI_KAFKA_PROVIDER
          ? OHI_KAFKA_QUERIES[`${show.toUpperCase()}_GROUPBY_${groupBy.toUpperCase()}`]  // NEW
          : DIM_QUERIES[`${show.toUpperCase()}_GROUPBY_${groupBy.toUpperCase()}`]
    : baseHealthQuery;

  // ... rest of the implementation ...
}
```

## 6. UI Component Updates

### File: `/common/components/home-table/index.tsx`

Update to handle OHI Kafka provider in table rendering, ensuring provider logos and names are displayed correctly.

### File: `/common/components/filter-bar/index.tsx`

Ensure filter bar supports OHI Kafka as a provider option.

### File: `/common/components/EntityNavigator/EntityNavigator.tsx`

Update entity navigator to handle OHI entity types in visualization.

### File: `/common/config/summary-config.ts`

```typescript
export const SUMMARY_CONFIG = [
  {
    groupId: 1,
    id: METRIC_IDS.THROUGHPUT_BY_CLUSTER,
    title: 'Incoming throughput by clusters (bytes/sec)',
    dimensions: {
      [MSK_PROVIDER]: { column: 1, row: 1, width: 12, height: 3 },
      [CONFLUENT_CLOUD_PROVIDER]: { column: 1, row: 1, width: 12, height: 3 },
      [OHI_KAFKA_PROVIDER]: { column: 1, row: 1, width: 12, height: 3 },  // NEW
    },
    prodivers: [MSK_PROVIDER, CONFLUENT_CLOUD_PROVIDER, OHI_KAFKA_PROVIDER],  // Updated
    chartType: 'viz.line',
  },
  // Update all other chart configurations similarly
];
```

## 7. Test Coverage Updates

### Create new test file: `/common/config/constants.ohi.spec.js`

```javascript
import {
  OHI_KAFKA_PROVIDER,
  OHI_HIDDEN_FILTERS_CONDITION,
  CLUSTER_COLUMNS_MAPPING,
} from './constants';

describe('OHI Kafka Provider Constants', () => {
  it('should have correct OHI provider name', () => {
    expect(OHI_KAFKA_PROVIDER).toBe('OHI Kafka');
  });

  it('should have correct OHI hidden filters', () => {
    expect(OHI_HIDDEN_FILTERS_CONDITION).toBe(
      "metricName like '%kafka%' AND instrumentation.provider = 'kafka'"
    );
  });

  it('should return correct cluster column mapping for OHI', () => {
    const result = CLUSTER_COLUMNS_MAPPING(false, OHI_KAFKA_PROVIDER);
    expect(result).toBe('`kafka.cluster.name` or `clusterName`');
  });
});
```

### Update existing test files

Update test files in:
- `/common/utils/query-utils.spec.js` - Add OHI query tests
- `/common/utils/data-utils.spec.js` - Add OHI data preparation tests
- `/common/hooks/use-fetch-entity-metrics/useFetchClusterMetrics.spec.js` - Add OHI metric fetching tests

## 8. E2E Test Updates

### File: `/e2e/tests/message-queues-home.spec.ts`

Add E2E tests for OHI Kafka provider selection and entity display.

## 9. Implementation Checklist

### Backend Entity Configuration
- [ ] Ensure New Relic backend recognizes ONHOSTKAFKACLUSTER, ONHOSTKAFKATOPIC, ONHOSTKAFKABROKER entity types
- [ ] Verify OHI integration sends proper entity metadata

### Metric Attribute Mapping
- [ ] Confirm OHI Kafka metric attribute names (kafka.cluster.name, kafka.topic, kafka.broker.id, etc.)
- [ ] Map OHI metrics to standard Kafka metrics used in the app

### UI Updates
- [ ] Add OHI Kafka logo/icon to assets if needed
- [ ] Update all dropdown menus to include OHI Kafka option
- [ ] Ensure charts and visualizations work with OHI data

### Query Optimization
- [ ] Test NRQL queries with actual OHI data
- [ ] Optimize query performance for large OHI deployments

### Testing
- [ ] Unit tests for all new OHI-specific functions
- [ ] Integration tests with mock OHI data
- [ ] E2E tests covering OHI workflows
- [ ] Manual testing with real OHI Kafka clusters

### Documentation
- [ ] Update README.md with OHI Kafka support
- [ ] Document OHI-specific configuration requirements
- [ ] Add OHI examples to CLAUDE.md

## 10. Migration Considerations

### Backward Compatibility
- All existing MSK and Confluent Cloud functionality must remain unchanged
- Filter persistence should handle new provider gracefully

### Performance
- Add appropriate indexes for OHI entity searches
- Consider pagination for large OHI deployments

### Error Handling
- Handle cases where OHI metrics might be missing
- Provide clear error messages for OHI-specific issues

### Data Migration
- No data migration required as OHI is a new integration
- Ensure clean separation between different provider data

### Rollback Strategy
- Feature flag to enable/disable OHI Kafka support
- Ability to hide OHI option without breaking existing functionality

## Appendix: OHI Kafka Metric Reference

### Common OHI Kafka Metrics
- `kafka.net.bytes_in.rate` - Incoming bytes per second
- `kafka.net.bytes_out.rate` - Outgoing bytes per second
- `kafka.topic.bytes_in.rate` - Topic incoming bytes per second
- `kafka.topic.bytes_out.rate` - Topic outgoing bytes per second
- `kafka.topic.messages_in.rate` - Messages in per second
- `kafka.controller.active.count` - Active controller count
- `kafka.partition.offline` - Offline partitions count
- `kafka.partition.underreplicated` - Under-replicated partitions

### Entity Attributes
- `kafka.cluster.name` - Cluster name
- `kafka.broker.id` - Broker ID
- `kafka.topic` - Topic name
- `instrumentation.provider` - Should be 'kafka' for OHI

This specification provides a comprehensive guide for implementing OHI Kafka support across the entire Message Queues application, maintaining consistency with existing patterns while accommodating OHI-specific requirements.
