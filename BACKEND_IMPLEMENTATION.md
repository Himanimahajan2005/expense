# Smart Expense Tracker - Backend & Database Implementation Guide

## Overview

This document provides comprehensive information about the backend and database implementation of the Smart Expense Tracker application.

## Project Structure

```
lib/
├── expense_model.dart              # Expense data model with serialization
├── split_model.dart                # Split and participant data models
├── database_helper.dart            # SQLite database operations
├── notification_service.dart       # Expense and split service logic
├── constants.dart                  # App-wide constants
├── services/
│   └── backend_api_service.dart   # High-level API interface
├── main.dart                       # App entry point
├── screens/                        # UI screens
└── widgets/                        # Reusable UI components
```

## Database Schema

### Tables

#### 1. **expenses**
```sql
CREATE TABLE expenses (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  amount REAL NOT NULL,
  category TEXT NOT NULL,
  description TEXT,
  date TEXT NOT NULL,
  created_at TEXT DEFAULT CURRENT_TIMESTAMP
)
```

**Columns:**
- `id`: Unique identifier
- `amount`: Expense amount (in decimal)
- `category`: Type of expense
- `description`: Additional details
- `date`: When the expense occurred
- `created_at`: Timestamp of record creation

#### 2. **splits**
```sql
CREATE TABLE splits (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  total_amount REAL NOT NULL,
  number_of_people INTEGER NOT NULL,
  date TEXT NOT NULL,
  created_at TEXT DEFAULT CURRENT_TIMESTAMP
)
```

**Columns:**
- `id`: Unique identifier
- `total_amount`: Total expense to split
- `number_of_people`: Number of participants
- `date`: When the split occurred
- `created_at`: Timestamp of record creation

#### 3. **split_participants**
```sql
CREATE TABLE split_participants (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  split_id INTEGER NOT NULL,
  person_name TEXT NOT NULL,
  amount_owed REAL NOT NULL,
  paid INTEGER DEFAULT 0,
  FOREIGN KEY(split_id) REFERENCES splits(id) ON DELETE CASCADE
)
```

**Columns:**
- `id`: Unique identifier
- `split_id`: Reference to parent split
- `person_name`: Name of participant
- `amount_owed`: Amount this person owes
- `paid`: Payment status (0 = unpaid, 1 = paid)

## Models

### ExpenseModel

Represents a single expense transaction.

```dart
ExpenseModel(
  amount: 50.0,
  category: 'Food & Dining',
  description: 'Lunch with friends',
  date: DateTime.now(),
)
```

**Methods:**
- `toJson()`: Convert to JSON
- `fromJson()`: Create from JSON
- `copyWith()`: Create modified copy

### SplitModel & SplitParticipant

Represent split transactions and participants.

```dart
SplitModel(
  totalAmount: 100.0,
  numberOfPeople: 4,
  participants: [
    SplitParticipant(personName: 'Alice', amountOwed: 25.0),
    SplitParticipant(personName: 'Bob', amountOwed: 25.0),
    // ... more participants
  ],
  date: DateTime.now(),
)
```

**Computed Properties:**
- `amountPerPerson`: Calculated per-person share
- `totalPaid`: Sum of paid amounts
- `totalUnpaid`: Sum of unpaid amounts

## Database Helper

The `DatabaseHelper` class provides low-level database operations using SQLite.

### Expense Operations

```dart
// Add expense
int expenseId = await dbHelper.insertExpense(expenseModel);

// Get all expenses
List<ExpenseModel> expenses = await dbHelper.getAllExpenses();

// Get by date range
List<ExpenseModel> rangeExpenses = await dbHelper.getExpensesByDateRange(
  startDate: DateTime(2024, 1, 1),
  endDate: DateTime(2024, 1, 31),
);

// Get by category
List<ExpenseModel> foodExpenses = await dbHelper.getExpensesByCategory('Food & Dining');

// Update expense
await dbHelper.updateExpense(modifiedExpenseModel);

// Delete expense
await dbHelper.deleteExpense(expenseId);

// Get statistics
double total = await dbHelper.getTotalExpenses();
Map<String, double> categoryWise = await dbHelper.getCategoryWiseSummary();
Map<String, double> monthly = await dbHelper.getMonthlySummary(2024, 1);
```

### Split Operations

```dart
// Create split
int splitId = await dbHelper.insertSplit(splitModel);

// Get all splits
List<SplitModel> splits = await dbHelper.getAllSplits();

// Get split by ID
SplitModel? split = await dbHelper.getSplitById(splitId);

// Update split
await dbHelper.updateSplit(modifiedSplitModel);

// Delete split
await dbHelper.deleteSplit(splitId);

// Mark participant as paid
await dbHelper.markParticipantAsPaid(participantId, true);

// Get split statistics
double totalSplits = await dbHelper.getTotalSplits();
```

### Utility Operations

```dart
// Export data
Map<String, dynamic> data = await dbHelper.exportData();

// Get statistics
Map<String, dynamic> stats = await dbHelper.getDatabaseStats();

// Clear all data
await dbHelper.clearAllData();

// Close database
await dbHelper.close();
```

## Expense Service

The `ExpenseService` class provides business logic for expenses and splits.

### Usage Examples

```dart
final expenseService = ExpenseService();

// Add expense
int id = await expenseService.addExpense(
  amount: 50.0,
  category: 'Food & Dining',
  description: 'Lunch',
  date: DateTime.now(),
);

// Get today's expenses
List<ExpenseModel> todayExpenses = await expenseService.getTodayExpenses();

// Get month expenses
List<ExpenseModel> monthExpenses = await expenseService.getMonthExpenses();

// Create split
int splitId = await expenseService.createSplit(
  totalAmount: 100.0,
  numberOfPeople: 4,
  participantNames: ['Alice', 'Bob', 'Charlie', 'Dave'],
  date: DateTime.now(),
);

// Get statistics
Map<String, dynamic> stats = await expenseService.getStatistics();

// Calculate average
double average = await expenseService.getAverageDailyExpense();
```

## Backend API Service

The `BackendAPIService` provides a high-level API interface with error handling using a generic `Result<T>` wrapper.

### Usage Examples

```dart
final api = BackendAPIService();

// Create expense with error handling
Result<int> result = await api.createExpense(
  amount: 50.0,
  category: 'Food & Dining',
  description: 'Lunch',
  date: DateTime.now(),
);

if (result.isSuccess) {
  int expenseId = result.data!;
  print('Expense created: $expenseId');
} else {
  print('Error: ${result.error}');
}

// Fetch all expenses
Result<List<ExpenseModel>> expensesResult = await api.fetchAllExpenses();
List<ExpenseModel> expenses = expensesResult.getOrDefault([]);

// Create split
Result<int> splitResult = await api.createSplit(
  totalAmount: 100.0,
  numberOfPeople: 4,
  participantNames: ['Alice', 'Bob', 'Charlie', 'Dave'],
  date: DateTime.now(),
);

// Get statistics
Result<Map<String, dynamic>> statsResult = await api.getStatistics();
Map<String, dynamic> stats = statsResult.getOrThrow();

// Search expenses
Result<List<ExpenseModel>> searchResult = await api.searchExpenses('lunch');
List<ExpenseModel> results = searchResult.data ?? [];

// Export data
Result<Map<String, dynamic>> exportResult = await api.exportData();
if (exportResult.isSuccess) {
  Map<String, dynamic> data = exportResult.data!;
  // Handle export
}
```

## Result<T> Wrapper

Generic result type for API responses with error handling:

```dart
class Result<T> {
  final T? data;
  final String? error;
  final bool isSuccess;

  // Get data or throw exception
  T getOrThrow();

  // Get data or return default value
  T getOrDefault(T defaultValue);

  // Map the result to another type
  Result<U> map<U>(U Function(T) mapper);
}
```

## Constants

### Expense Categories

```dart
ExpenseCategories.food              // 'Food & Dining' 🍽️
ExpenseCategories.transport         // 'Transport' 🚗
ExpenseCategories.entertainment     // 'Entertainment' 🎬
ExpenseCategories.utilities         // 'Utilities' 💡
ExpenseCategories.shopping          // 'Shopping' 🛍️
ExpenseCategories.healthcare        // 'Healthcare' ⚕️
ExpenseCategories.education         // 'Education' 📚
ExpenseCategories.other             // 'Other' 💳

// Get all categories
List<String> categories = ExpenseCategories.allCategories;

// Get emoji for category
String emoji = ExpenseCategories.getCategoryEmoji('Food & Dining'); // 🍽️
```

### Colors

```dart
AppColors.primary       // #4D96EE (Bright Blue)
AppColors.secondary     // #34C3FF (Light Cyan)
AppColors.success       // #1BBE74 (Green)
AppColors.danger        // #D84D5C (Red)
AppColors.border        // #D8E4F3 (Light Gray-Blue)
AppColors.background    // #F4F7FB (Very Light Blue)
AppColors.lightBg       // #F9FBFE (Very Light Blue)
```

## Dependencies

Add these to your `pubspec.yaml`:

```yaml
dependencies:
  flutter:
    sdk: flutter
  sqflite: ^2.3.0              # SQLite database
  path: ^1.8.3                 # Path utilities
  intl: ^0.19.0                # Internationalization
  provider: ^6.1.0             # State management (optional)
  cupertino_icons: ^1.0.8      # iOS icons
```

## Installation Instructions

1. **Update pubspec.yaml**
   ```bash
   flutter pub get
   ```

2. **Run the app**
   ```bash
   flutter run
   ```

3. **Build for production**
   ```bash
   flutter build apk    # Android
   flutter build ios    # iOS
   flutter build web    # Web
   ```

## API Reference

### Add Expense
- **Method**: POST
- **Endpoint**: `BackendAPIService.createExpense()`
- **Parameters**:
  - `amount` (double): Expense amount
  - `category` (String): Category name
  - `description` (String): Optional description
  - `date` (DateTime): Transaction date
- **Returns**: `Result<int>` (Expense ID)

### Get Expenses
- **Method**: GET
- **Endpoint**: `BackendAPIService.fetchAllExpenses()`
- **Returns**: `Result<List<ExpenseModel>>`

### Create Split
- **Method**: POST
- **Endpoint**: `BackendAPIService.createSplit()`
- **Parameters**:
  - `totalAmount` (double): Total amount to split
  - `numberOfPeople` (int): Number of participants
  - `participantNames` (List<String>): Names of participants
  - `date` (DateTime): Split date
- **Returns**: `Result<int>` (Split ID)

### Get Statistics
- **Method**: GET
- **Endpoint**: `BackendAPIService.getStatistics()`
- **Returns**: `Result<Map<String, dynamic>>`

### Search Expenses
- **Method**: GET
- **Endpoint**: `BackendAPIService.searchExpenses()`
- **Parameters**:
  - `query` (String): Search keyword
- **Returns**: `Result<List<ExpenseModel>>`

## Error Handling

All API methods return a `Result<T>` that contains either success data or an error message:

```dart
final result = await api.createExpense(...);

if (result.isSuccess) {
  // Handle success
  int id = result.data!;
} else {
  // Handle error
  print(result.error); // Error message
}
```

## Future Enhancements

- [ ] Cloud synchronization (Firebase)
- [ ] Push notifications for reminders
- [ ] Receipt image storage
- [ ] Advanced analytics
- [ ] Multi-currency support
- [ ] Export to PDF/CSV
- [ ] Social sharing features
- [ ] Recurring expenses

## Troubleshooting

### Database not initializing
- Ensure `sqflite` package is installed
- Check file permissions on device/emulator
- Clear app data and rebuild

### Null pointer exceptions
- Always check `Result.isSuccess` before accessing `Result.data`
- Use `getOrDefault()` for safe access

### Performance issues
- Add indexes to frequently queried columns
- Implement pagination for large datasets
- Consider caching frequently accessed data

## Support & Documentation

For more information:
- [Flutter Documentation](https://flutter.dev/docs)
- [SQLite Documentation](https://sqlite.org/docs.html)
- [Dart Documentation](https://dart.dev/guides)
