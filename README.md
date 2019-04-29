# Forms
Proof of Concept filtering on dynamic forms

For more details, see blog post at: https://maxdemarzi.com/2019/04/28/filtering-connected-dynamic-forms/

See data.cypher in folder for sample data.

Create Indexes:

	CREATE INDEX ON :Project(id);
	CREATE INDEX ON :Field(id);


	  // Start with everyone on the Project
	  MATCH (prj:Project {id:'Project 1'})-[:INCLUDES]->(p:Person),
	  // Gather all of their relevant responses
      (p)-[:HAS_RESPONSE]-(r:Response)-[:OF_FORM]->(form:Form)
	  WHERE form.id IN ['Form 1', 'Form 2', 'Form 3']
	  WITH p, COLLECT(r) AS responses
	  // First Filter (Field 1, value 1)
	  // Gather all responses to Field 1 from available responses
	  MATCH (r2)-[:LINKED*0..]-(r)-[hv:HAS_VALUE]->(f:Field)
	  WHERE f.id = 'Field 1' AND r IN responses
	  WITH p, responses, hv, COLLECT(DISTINCT r2) AS found, COLLECT(hv) AS hvs
	  // Make sure we have at least 1 valid answer from the person
	  WHERE ANY (x IN hvs WHERE x.value = 1)
	  // Remove responses and their chains for invalid values
	  WITH p, [x IN responses WHERE NOT x IN CASE WHEN hv.value = 1 THEN [] ELSE found END] AS responses
	  // Second Filter (Field 2, value 1) -- same as first filter
	  MATCH (r2)-[:LINKED*0..]-(r)-[hv:HAS_VALUE]->(f:Field)
	  WHERE f.id = 'Field 2' AND r IN responses
	  WITH p, responses, hv, COLLECT(DISTINCT r2) AS found, COLLECT(hv) AS hvs
	  WHERE ANY (x IN hvs WHERE x.value = 1)
	  WITH p, [x IN responses WHERE NOT x IN CASE WHEN hv.value = 1 THEN [] ELSE found END] AS responses
	  // Third Filter (Field 4, value 2) -- same as third filter
	  MATCH (r2)-[:LINKED*0..]-(r)-[hv:HAS_VALUE]->(f:Field)
	  WHERE f.id = 'Field 4' AND r IN responses
	  WITH p, responses, hv, COLLECT(DISTINCT r2) AS found, COLLECT(hv) AS hvs
	  WHERE ANY (x IN hvs WHERE x.value = 2)
	  WITH p, [x IN responses WHERE NOT x IN CASE WHEN hv.value = 2 THEN [] ELSE found END] AS responses
	  UNWIND responses AS response
	  // With valid left over responses, gather values for the form we want
	  MATCH (response)-[hv:HAS_VALUE]->(f:Field)<-[:HAS_FIELD]-(form:Form)
	  WHERE form.id = "Form 3" 
	  RETURN p, response, f, hv, form
	  ORDER BY p, response.at, f.id

