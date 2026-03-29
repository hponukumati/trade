# Trade Data Import System

Rust Endpoints are used in our "[exiobase/tradeflow](https://model.earth/exiobase/tradeflow/)" pull into SQL

Alternative (but not finalized):

.NET 8.0 web application for importing CSV trade data into PostgreSQL database.

## Quick Start

### 1. Prerequisites
- .NET 8.0 SDK
- PostgreSQL database (Azure or local)
- CSV files from [trade-data repository](https://github.com/IndustryDB/trade-data)

### 2. Setup

1. **Clone the repository**
```bash
git clone https://github.com/modelearth/trade.git
cd trade
```

2. **Create .env file**
```bash
cp .env.example .env
```

3. **Edit .env with your settings**
```env
DATABASE_HOST=your-postgres-server.postgres.database.azure.com
DATABASE_NAME=exiobase
DATABASE_USER=postgresadmin
DATABASE_PASSWORD=your_password
DATABASE_PORT=5432

TRADE_DATA_REPO_PATH=../trade-data
```

4. **Run database migrations**
```bash
# Connect to your PostgreSQL database and run these scripts in order:
# 1. IndustryDB/DB Scripts/Postgres/001_CreateTradeTable.sql
# 2. IndustryDB/DB Scripts/Postgres/002_CreateAdditionalTables.sql
# 3. IndustryDB/DB Scripts/Postgres/003_CreateStoredProcs.sql
```

5. **Run the application**
```bash
cd IndustryDB
dotnet restore
dotnet run
```

6. **Access the import interface**
- Navigate to: `https://localhost:5001/TradeImport`
- Select year (2019 or 2022)
- Click "Create Database" to start import

## Architecture

### Data Flow
```
405 CSV files (15 countries × 3 tradeflows × 9 file types)
    ↓
9 PostgreSQL Tables (consolidated)
    ↓
REST API (background jobs)
    ↓
Web UI (progress tracking)
```

### Components
- **CsvImportService** - Reads and parses CSV files
- **TradeDataRepository** - Database operations (Dapper + Npgsql)
- **TradeImportController** - REST API endpoints
- **Views/TradeImport** - User interface

### Database Tables
1. `public.trade` - Main trade flow data
2. `public.trade_employment` - Employment impacts
3. `public.trade_factor` - Production factors

These are redundant, so we're not adding as SQL tables:

4. `public.trade_impact` - Economic impacts
5. `public.trade_material` - Material flows
6. `public.trade_resource` - Resource usage
7-9. `public.bea_table1/2/3` - BEA (Bureau of Economic Analysis) data

## API Endpoints

### Import Data
```http
POST /api/tradeimport/create-database
Content-Type: application/json

{
  "year": 2022,
  "countries": ["US", "IN"],  // null for all countries
  "clearExistingData": false
}
```

### Get Import Status
```http
GET /api/tradeimport/status/{jobId}
```

### Get Statistics
```http
GET /api/tradeimport/statistics/{year}
```

### Test Connection
```http
GET /api/tradeimport/test-connection
```

## CSV File Structure

```
trade-data/
└── year/
    ├── 2019/
    │   └── US/
    │       ├── imports/
    │       │   ├── trade.csv
    │       │   ├── trade_employment.csv
    │       │   ├── trade_factor.csv
    │       │   ├── trade_impact.csv
    │       │   ├── trade_material.csv
    │       │   ├── trade_resource.csv
    │       │   └── (3 BEA files)
    │       ├── exports/
    │       │   └── (same 9 files)
    │       └── domestic/
    │           └── (same 9 files)
    └── 2022/
        └── (same structure)
```

## Configuration (.env)

| Variable | Description | Example |
|----------|-------------|---------|
| `DATABASE_HOST` | PostgreSQL server hostname | `localhost` or Azure server |
| `DATABASE_NAME` | Database name | `exiobase` |
| `DATABASE_USER` | Database username | `postgresadmin` |
| `DATABASE_PASSWORD` | Database password | `YourSecurePassword!` |
| `DATABASE_PORT` | PostgreSQL port | `5432` |
| `TRADE_DATA_REPO_PATH` | Path to CSV files | `../trade-data` |
| `BATCH_SIZE` | Batch insert size | `1000` |

## Troubleshooting

### "TRADE_DATA_REPO_PATH environment variable not set"
- Ensure `.env` file exists in project root
- Verify `TRADE_DATA_REPO_PATH` is set correctly
- Check that the path exists and contains CSV files

### "Database connection failed"
- Test connection using "Test Connection" button
- Verify PostgreSQL credentials in `.env`
- Ensure database server is accessible
- Check firewall rules (especially for Azure)

### "No CSV files found"
- Verify `TRADE_DATA_REPO_PATH` points to correct location
- Ensure CSV files exist in expected structure: `year/{year}/{country}/{tradeflow}/*.csv`
- Check that you have the trade-data repository cloned

## Performance

### Import Times
- **Per country**: ~30 minutes (3 tradeflows × 10 min each)
- **Full year (15 countries)**: ~7.5 hours
- **Batch size**: 1000 records per INSERT

### Optimization Tips
- Use SSD for CSV file storage
- Increase `BATCH_SIZE` for faster imports (test first)
- Disable database indexes during bulk import, rebuild after
- Use local PostgreSQL for development (faster than Azure)

## Testing

### Run Tests
```bash
# Run all tests
cd IndustryDB.Tests
dotnet test

# Run unit tests only
dotnet test --filter "Category!=Integration"

# Run integration tests only
dotnet test --filter "Category=Integration"
```

### Test Coverage
- ✅ CsvImportService (file discovery, parsing, validation)
- ✅ TradeDataRepository (SQL generation, database operations)
- ✅ TradeImportController (API endpoints, validation)
- ✅ Integration tests (end-to-end import workflow)

See [IndustryDB.Tests/README.md](IndustryDB.Tests/README.md) for detailed testing documentation.

## Development

### Project Structure
```
IndustryDB/
├── Controllers/
│   └── TradeImportController.cs
├── Models/Data/
│   ├── Trade.cs
│   ├── TradeImportRecord.cs
│   ├── ImportStatus.cs
│   └── DatabaseCreationRequest.cs
├── Services/
│   ├── CsvImportService.cs
│   └── TradeDataRepository.cs
├── Views/TradeImport/
│   └── Index.cshtml
└── DB Scripts/Postgres/
    ├── 001_CreateTradeTable.sql
    ├── 002_CreateAdditionalTables.sql
    └── 003_CreateStoredProcs.sql

IndustryDB.Tests/
├── Controllers/
│   └── TradeImportControllerTests.cs
├── Services/
│   ├── CsvImportServiceTests.cs
│   └── TradeDataRepositoryTests.cs
└── Integration/
    └── CsvImportIntegrationTests.cs
```

### Technologies
- .NET 8.0 (ASP.NET Core MVC)
- PostgreSQL (Npgsql driver)
- Dapper (micro-ORM)
- CsvHelper (CSV parsing)
- dotenv.net (.env file support)
- Bootstrap 5 (UI)
- xUnit, Moq, FluentAssertions (Testing)

## Documentation

- **Complete Documentation**: [docs/](docs/)
  - [Getting Started Guide](docs/guides/GETTING_STARTED.md)
  - [System Architecture](docs/ARCHITECTURE.md)
  - [Testing Guide](docs/testing/TESTING_GUIDE.md)
  - [Complete Test Flow](docs/guides/COMPLETE_TEST_FLOW.md)
- **GitHub Issue**: [#30 - Generate trade flow SQL](https://github.com/modelearth/trade/issues/30)
- **CSV Data Source**: [IndustryDB/trade-data](https://github.com/IndustryDB/trade-data)
- **Trade Visualization**: https://model.earth/profile/footprint/

## License

See repository license.

## Support

For issues or questions, please open an issue on the GitHub repository.
