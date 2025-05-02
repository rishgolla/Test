Handling Dynamic Schema with GraphQL

Core Idea:

● The GraphQL schema will define a type representing a log entry.
● This type will have fields for the core, expected columns you anticipate being present in most/all version-specific tables like model_id, model_version_id etc.
● It will also have a field additional_data of type strawberry.JSON to hold all other columns found in that specific table row, accommodating the dynamic nature.
● The API query will require the modelVersionId (or a similar identifier used to construct the table name) as an argument.
● The resolver logic will safely construct the table name, query SELECT * from that table, and map the results into the GraphQL type, placing unknown columns into the additional_data field.
Sample Schema Design:

 
import strawberry
import datetime
from typing import List, Optional, Dict, Any
 
# strawberry.JSON allows returning arbitrary JSON structures
JSON = strawberry.scalars.JSON
 
@strawberry.type
class LogEntry:
  """Represents a single log entry from a model version table"""
  # Expected Fields example
  interaction_id: strawberry.ID
  request_timestamp: Optional[datetime.datetime]
  response_timestamp: Optional[datetime.datetime]
  prompt: Optional[str]
  output_text: Optional[str]
  input_token_count: Optional[int]
  output_token_count: Optional[int]
  finish_reason: Optional[str]
  user_id: Optional[str]
   # Field for Dynamic Columns
  additional_data: Optional[JSON]
 
@strawberry.type
class Query:
  @strawberry.field
  async def get_logs_by_version(
      self,
      info: strawberry.Info,
      model_version_id: str,
      limit: int = 100,
      offset: int = 0,
 
  ) -> List[LogEntry]:
      """
      Fetches logs for a specific model version.
      Requires the model_version_id to identify the target table.
      """
      db_pool = info.context.get("db_pool")
      if not db_pool:
          raise ValueError("Database connection pool not found in context")
 
      # Option 1: Strict Pattern Validation (Example)
      import re
      if not re.fullmatch(r'^[a-zA-Z0-9_.-]+$', model_version_id):
           raise ValueError("Invalid model_version_id format")
      # Prepend a fixed prefix to prevent accessing unintended tables
      table_name = f"log_{model_version_id}" # Adjust prefix as needed
 
      # Option 2: Allowlist/Lookup (Not sure if this is optimal)
      # allowed_tables = get_allowed_tables_from_config_or_db()
      # if model_version_id not in allowed_tables:
      #     raise ValueError(f"Table for model version '{model_version_id}' not allowed or not found.")
      # table_name = allowed_tables[model_version_id] # Assuming lookup returns actual table name
 
      # --- Define Core Columns (match LogEntry fields) ---
      # These are the columns we will explicitly map
      core_columns = {
          "interaction_id",
          "request_timestamp",
          "response_timestamp",
          "prompt",
          "output_text",
          "input_token_count",
          "output_token_count",
          "finish_reason",
          "user_id",
      }
 
      results = []
      try:
          async with db_pool.acquire() as connection:
              # Fetch all columns using SELECT *. Parameterize WHERE clause values.
              query = f'SELECT * FROM "{table_name}" ORDER BY request_timestamp DESC LIMIT $1 OFFSET $2' # Example for PostgreSQL
 
              rows = await connection.fetch(query, limit, offset) # Use fetch for asyncpg
 
              for row in rows:
                  # Convert row to a dictionary (asyncpg rows are Record objects)
                  row_dict = dict(row)
 
                  core_data = {}
                  additional_data = {}
 
                  for col_name, value in row_dict.items():
                      if col_name in core_columns:
                          core_data[col_name] = value
                      else:
                          additional_data[col_name] = value
 
                  # Handle potential type mismatches or nulls gracefully
                  entry = LogEntry(
                      interaction_id=core_data.get("interaction_id"),
                      request_timestamp=core_data.get("request_timestamp"),
                      response_timestamp=core_data.get("response_timestamp"),
                      prompt=core_data.get("prompt"),
                      output_text=core_data.get("output_text"),
                      input_token_count=core_data.get("input_token_count"),
                      output_token_count=core_data.get("output_token_count"),
                      finish_reason=core_data.get("finish_reason"),
                      user_id=core_data.get("user_id"),
                      additional_data=additional_data if additional_data else None,
                  )
                  results.append(entry)
 
      except Exception as e:
          print(f"Error fetching data for table {table_name}: {e}")
          # Need to handle errors based on the usecase like table not exist etc
          raise RuntimeError(f"Could not retrieve data for model version {model_version_id}. Error: {e}")
 
      return results
 
# Create the GraphQL schema
schema = strawberry.Schema(query=Query)
 

Fast Api:
 
from fastapi import FastAPI, Depends
from strawberry.fastapi import GraphQLRouter
from schema import schema
import asyncpg
import os
 
 
DB_POOL = None
 
async def get_db_pool():
  global DB_POOL
  if DB_POOL is None:
      try:
         
          DB_POOL = await asyncpg.create_pool(
              user=os.getenv("RDS_USER"),
              password=os.getenv("RDS_PASSWORD"),
              database=os.getenv("RDS_DB"),
              host=os.getenv("RDS_HOST"),
              port=os.getenv("RDS_PORT", 5432)
          )
      except Exception as e:
 
          print(f"FATAL: Could not connect to database pool: {e}")
          # Depending on strategy, we cann exit or prevent startup
          raise RuntimeError("Database connection failed") from e
  return DB_POOL
 
 
# This function gets the pool and adds it to the Strawberry context
async def get_context(db_pool = Depends(get_db_pool)):
  return {
      "db_pool": db_pool
  }
 
# FastAPI App
graphql_app = GraphQLRouter(
  schema,
  context_getter=get_context
)
 
app = FastAPI()
app.include_router(graphql_app, prefix="/graphql")
 
@app.on_event("startup")
async def startup_event():
  # Initialize the pool on startup
  await get_db_pool()
  print("Database pool created.")
 
@app.on_event("shutdown")
async def shutdown_event():
  # Close the pool on shutdown
  global DB_POOL
  if DB_POOL:
      await DB_POOL.close()
      print("Database pool closed.")
 
@app.get("/")
async def root():
  return {"message": "Navigate to /graphql for the API"}
 

Considerations:
● To avoid dynamically generating table names from user input for security reasons, we can use a strict allowlist, regular expression pattern matching, or look up the valid table name from a configuration/metadata table based on the input ID.
● Dynamically created tables and especially dynamic columns within those tables conflict with statically typed GraphQL schemas
● This approach prioritizes flexibility over potential query optimization for specific columns. We might need to index core columns
 
Compatibility with Federation:
 
● Data inside additional_data: JSON is opaque to federation; it cannot be directly linked or required by other subgraphs.
● Limited Federation Capabilities for Dynamic Data: You cannot define fields in other subgraphs that directly @require or @provide data from within the additional_data blob of a LogEntry. Federation operates on the defined GraphQL fields, not the opaque content of a JSON scalar.
● Discoverability: The dynamic columns hidden within additional_data are not discoverable via the standard supergraph introspection. Clients need out-of-band knowledge of what might be inside additional_data for a given model_version_id.
● Operational Complexity: Managing a subgraph where the data source changes based on query arguments adds complexity compared to typical subgraphs that query fixed tables/sources.
 