Show data usage statistics for GEOME projects

Navigate to https://biocodellc.github.io/geome-statistics/ to view the latest data.

Run following script and save to: `data/statistics.csv` and then upload to github;

```
SELECT
  p.id AS "projectId", p.project_title AS "projectTitle", p.latest_data_modification as "latestDataModification", p.description AS "description", p.public AS "public",
  p.discoverable AS "discoverable",
  p.principal_investigator AS "principalInvestigator",
  p.principal_investigator_affiliation AS "principalInvestigatorAffiliation",
  p.project_contact AS "projectContact",
  p.project_contact_email AS "projectContactEmail",
  p.publication_guid AS "publicationGuid",
  p.project_data_guid AS "projectDataGuid",
  p.recommended_citation AS "recommendedCitation",
  p.license AS "license",
  p.localcontexts_id AS "localcontextsId",
  p.permit_guid AS "permitGuid",
  pc.id AS "configId", pc.name AS "configName", pc.description AS "configDescription", pc.network_approved AS "configNetworkApproved",
  u.id AS "userId", u.username AS "username", u.email AS "email"
  , COALESCE(Event.ct, 0) as "EventCount"
, COALESCE(Sample.ct, 0) as "SampleCount"
, COALESCE(Diagnostics.ct, 0) as "DiagnosticsCount"
, COALESCE(Tissue.ct, 0) as "TissueCount"
, COALESCE(Sample_Photo.ct, 0) as "Sample_PhotoCount"
, COALESCE(Event_Photo.ct, 0) as "Event_PhotoCount"
, COALESCE(fastaSequence.ct, 0) as "fastaSequenceCount"
, COALESCE(fastqMetadata.ct, 0) as "fastqMetadataCount"

FROM projects AS p LEFT JOIN users AS u on p.user_id = u.id LEFT JOIN project_configurations AS pc on p.config_id = pc.id LEFT JOIN (
  SELECT project_id, count(*) as ct FROM network_1.Event LEFT JOIN expeditions e ON e.id = network_1.Event.expedition_id GROUP BY e.project_id
) AS Event on Event.project_id = p.id
LEFT JOIN (
  SELECT project_id, count(*) as ct FROM network_1.Sample LEFT JOIN expeditions e ON e.id = network_1.Sample.expedition_id GROUP BY e.project_id
) AS Sample on Sample.project_id = p.id
LEFT JOIN (
  SELECT project_id, count(*) as ct FROM network_1.Diagnostics LEFT JOIN expeditions e ON e.id = network_1.Diagnostics.expedition_id GROUP BY e.project_id
) AS Diagnostics on Diagnostics.project_id = p.id
LEFT JOIN (
  SELECT project_id, count(*) as ct FROM network_1.Tissue LEFT JOIN expeditions e ON e.id = network_1.Tissue.expedition_id GROUP BY e.project_id
) AS Tissue on Tissue.project_id = p.id
LEFT JOIN (
  SELECT project_id, count(*) as ct FROM network_1.Sample_Photo LEFT JOIN expeditions e ON e.id = network_1.Sample_Photo.expedition_id GROUP BY e.project_id
) AS Sample_Photo on Sample_Photo.project_id = p.id
LEFT JOIN (
  SELECT project_id, count(*) as ct FROM network_1.Event_Photo LEFT JOIN expeditions e ON e.id = network_1.Event_Photo.expedition_id GROUP BY e.project_id
) AS Event_Photo on Event_Photo.project_id = p.id
LEFT JOIN (
  SELECT project_id, count(*) as ct FROM network_1.fastaSequence LEFT JOIN expeditions e ON e.id = network_1.fastaSequence.expedition_id GROUP BY e.project_id
) AS fastaSequence on fastaSequence.project_id = p.id
LEFT JOIN (
  SELECT project_id, count(*) as ct FROM network_1.fastqMetadata LEFT JOIN expeditions e ON e.id = network_1.fastqMetadata.expedition_id GROUP BY e.project_id
) AS fastqMetadata on fastqMetadata.project_id = p.id
 WHERE (p.discoverable = true OR p.public = true OR p.id in (SELECT project_id from user_projects AS up where up.user_id = 0)) ORDER BY lower(p.project_title)
```

Run following script and save to: `data/configstatistics.csv` and then upload to github:

```
SELECT
    pc.id AS config_id,
    -- Conditionally choose project title or config name
    CASE 
        WHEN pc.network_approved = true THEN pc.name
        ELSE p.project_title
    END AS name,
    STRING_AGG(DISTINCT entity->>'worksheet', ', ') AS worksheets,
    STRING_AGG(DISTINCT entity->>'uniqueAcrossProject', ', ') AS uniqueAcrossProject,
    pc.network_approved,
    MAX(p.modified) AS latest_modified -- Get the max date modified
    --COALESCE(SUM(Sample.ct), 0) AS sample_count -- Aggregate sample count from the subquery
FROM projects p
JOIN project_configurations pc ON p.config_id = pc.id
CROSS JOIN LATERAL jsonb_array_elements(pc.config->'entities') AS entity
LEFT JOIN (
    SELECT e.project_id, COUNT(DISTINCT s.id) AS ct
    FROM network_1.Sample s
    LEFT JOIN expeditions e ON e.id = s.expedition_id
    GROUP BY e.project_id
) Sample ON Sample.project_id = p.id
WHERE entity->>'conceptAlias' IN ('Event', 'Sample', 'Tissue')
GROUP BY
    pc.id,
    CASE 
        WHEN pc.network_approved = true THEN pc.name
        ELSE p.project_title
    END,       
    pc.network_approved
HAVING 
    COALESCE(SUM(Sample.ct), 0) > 0 -- Only include rows with a sample count greater than 0
ORDER BY
    pc.network_approved DESC,
    latest_modified DESC,
    pc.id;

```


