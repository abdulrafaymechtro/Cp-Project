// ============================================
// CORPORATE EXPENSE TRACKER SYSTEM
// WITH ESP32 HARDWARE KEY AUTHENTICATION
// For Dev-C++ (Windows) + ESP32 on Serial Port
// ============================================

#include <iostream>
#include <fstream>
#include <sstream>
#include <string>
#include <vector>
#include <algorithm>
#include <cstdlib>
#include <iomanip>
#include <windows.h>   // Required for Serial Port (ESP32)

using namespace std;

// ============================================
// ESP32 CONFIGURATION
// ============================================
// *** CHANGE "COM3" BELOW TO YOUR ESP32 PORT ***
// Example: COM4, COM5, COM6 etc.
// Check Device Manager -> Ports to find your port
#define ESP32_COM_PORT  "\\\\.\\COM7"
#define ESP32_BAUD_RATE CBR_115200
#define SECRET_KEY      "ACCESS_123"   // Must match ESP32 code

// ============================================
// FUNCTION: ESP32 AUTHENTICATION
// Returns true if correct key received from ESP32
// ============================================
bool authenticateESP32() {

    cout << "\n========================================\n";
    cout << "   HARDWARE AUTHENTICATION REQUIRED\n";
    cout << "========================================\n";
    cout << "Connecting to ESP32 on " << ESP32_COM_PORT << "...\n";

    // --- Step 1: Open Serial Port ---
    HANDLE hSerial = CreateFile(
        ESP32_COM_PORT,
        GENERIC_READ | GENERIC_WRITE,
        0,
        NULL,
        OPEN_EXISTING,
        FILE_ATTRIBUTE_NORMAL,
        NULL
    );

    if (hSerial == INVALID_HANDLE_VALUE) {
        cout << "\n[AUTH FAILED] ESP32 not found on " << ESP32_COM_PORT << "!\n";
        cout << "Please connect your ESP32 and try again.\n";
        cout << "Tip: Check Device Manager to find the correct COM port.\n";
        return false;
    }

    cout << "[OK] Serial port opened.\n";

    // --- Step 2: Configure Serial Parameters ---
    DCB dcbSerialParams = { 0 };
    dcbSerialParams.DCBlength = sizeof(dcbSerialParams);

    if (!GetCommState(hSerial, &dcbSerialParams)) {
        cout << "\n[AUTH FAILED] Could not get serial port state!\n";
        CloseHandle(hSerial);
        return false;
    }

    dcbSerialParams.BaudRate = ESP32_BAUD_RATE;
    dcbSerialParams.ByteSize = 8;
    dcbSerialParams.StopBits = ONESTOPBIT;
    dcbSerialParams.Parity   = NOPARITY;

    if (!SetCommState(hSerial, &dcbSerialParams)) {
        cout << "\n[AUTH FAILED] Could not set serial port parameters!\n";
        CloseHandle(hSerial);
        return false;
    }

    // --- Step 3: Set Read Timeout (3 seconds) ---
    COMMTIMEOUTS timeouts = { 0 };
    timeouts.ReadIntervalTimeout         = 50;
    timeouts.ReadTotalTimeoutConstant    = 3000;  // 3 second total timeout
    timeouts.ReadTotalTimeoutMultiplier  = 10;
    timeouts.WriteTotalTimeoutConstant   = 1000;
    timeouts.WriteTotalTimeoutMultiplier = 10;
    SetCommTimeouts(hSerial, &timeouts);

    // --- Step 4: Give ESP32 time to boot (if freshly connected) ---
    Sleep(1500);

    // --- Step 5: Send 'R' to request the key ---
    DWORD bytesWritten;
    char request = 'R';

    if (!WriteFile(hSerial, &request, 1, &bytesWritten, NULL)) {
        cout << "\n[AUTH FAILED] Could not send request to ESP32!\n";
        CloseHandle(hSerial);
        return false;
    }

    cout << "Request sent to ESP32. Waiting for key...\n";

    // --- Step 6: Read the key response ---
    char szBuff[11] = { 0 };  // 10 bytes + null terminator
    DWORD bytesRead = 0;

    if (!ReadFile(hSerial, szBuff, 10, &bytesRead, NULL) || bytesRead == 0) {
        cout << "\n[AUTH FAILED] No response from ESP32 (timeout)!\n";
        cout << "Make sure the ESP32 is powered and running the auth sketch.\n";
        CloseHandle(hSerial);
        return false;
    }

    szBuff[bytesRead] = '\0';  // Null-terminate whatever we received

    CloseHandle(hSerial);

    cout << "Response received.\n";

    // --- Step 7: Validate the key ---
    string receivedKey(szBuff);

    // Trim any trailing whitespace/newlines just in case
    // Using index instead of .back() for Dev-C++ 5.11 compatibility
    while (receivedKey.size() > 0) {
        char lastChar = receivedKey[receivedKey.size() - 1];
        if (lastChar == '\r' || lastChar == '\n' || lastChar == ' ') {
            receivedKey.erase(receivedKey.size() - 1, 1);
        } else {
            break;
        }
    }

    if (receivedKey == SECRET_KEY) {
        cout << "\n[AUTH SUCCESS] ESP32 key verified!\n";
        cout << "Access granted. Loading system...\n";
        Sleep(800);
        return true;
    } else {
        cout << "\n[AUTH FAILED] Invalid key received: \"" << receivedKey << "\"\n";
        cout << "Access denied. Unauthorized device.\n";
        return false;
    }
}

// ============================================
// FUNCTION 0: INITIALIZE DATABASE
// ============================================
void initializeDatabase() {

    ifstream checkFile("expenses.csv");

    if (!checkFile.good()) {

        ofstream file("expenses.csv");

        file << "ExpID,EmpID,EmpName,Category,Amount,Date,Status\n";

        // Adding initial 100 records
        file << "EXP_101,EMP_001,Ahmed_Khan,Food,3200,2025-02-15,Pending\n";
        file << "EXP_102,EMP_002,Fatima_Ali,Equipment,5000,2025-03-20,Rejected\n";
        file << "EXP_103,EMP_003,Hassan_Malik,Utilities,7800,2025-04-05,Approved\n";
        file << "EXP_104,EMP_004,Zainab_Hassan,Maintenance,12000,2025-05-12,Pending\n";
        file << "EXP_105,EMP_005,Bilal_Ahmed,Travel,500,2025-06-18,Rejected\n";
        file << "EXP_106,EMP_006,Hana_Rashid,Food,9500,2025-01-10,Approved\n";
        file << "EXP_107,EMP_007,Omar_Ibrahim,Equipment,4200,2025-02-15,Pending\n";
        file << "EXP_108,EMP_008,Leila_Mohamed,Utilities,6700,2025-03-20,Rejected\n";
        file << "EXP_109,EMP_009,Karim_Hassan,Maintenance,2300,2025-04-05,Approved\n";
        file << "EXP_110,EMP_010,Amina_Samir,Travel,8900,2025-05-12,Pending\n";
        file << "EXP_111,EMP_011,Salim_Farooq,Food,11500,2025-06-18,Rejected\n";
        file << "EXP_112,EMP_012,Noor_Abdullah,Equipment,3700,2025-01-10,Approved\n";
        file << "EXP_113,EMP_013,Jamal_Yousuf,Utilities,15000,2025-02-15,Pending\n";
        file << "EXP_114,EMP_014,Rima_Karim,Maintenance,800,2025-03-20,Rejected\n";
        file << "EXP_115,EMP_015,Faris_Najib,Travel,1500,2025-04-05,Approved\n";
        file << "EXP_116,EMP_016,Dana_Hassan,Food,3200,2025-05-12,Pending\n";
        file << "EXP_117,EMP_017,Sami_Rashid,Equipment,5000,2025-06-18,Rejected\n";
        file << "EXP_118,EMP_018,Layla_Noor,Utilities,7800,2025-01-10,Approved\n";
        file << "EXP_119,EMP_019,Adel_Samir,Maintenance,12000,2025-02-15,Pending\n";
        file << "EXP_120,EMP_020,Maryam_Ibrahim,Travel,500,2025-03-20,Rejected\n";
        file << "EXP_121,EMP_021,Rashid_Ali,Food,9500,2025-04-05,Approved\n";
        file << "EXP_122,EMP_022,Yasmin_Karim,Equipment,4200,2025-05-12,Pending\n";
        file << "EXP_123,EMP_023,Waleed_Farooq,Utilities,6700,2025-06-18,Rejected\n";
        file << "EXP_124,EMP_024,Huda_Mohamed,Maintenance,2300,2025-01-10,Approved\n";
        file << "EXP_125,EMP_025,Tariq_Hassan,Travel,8900,2025-02-15,Pending\n";
        file << "EXP_126,EMP_026,Samira_Yousuf,Food,11500,2025-03-20,Rejected\n";
        file << "EXP_127,EMP_027,Nadir_Abdullah,Equipment,3700,2025-04-05,Approved\n";
        file << "EXP_128,EMP_028,Lina_Salim,Utilities,15000,2025-05-12,Pending\n";
        file << "EXP_129,EMP_029,Majid_Rashid,Maintenance,800,2025-06-18,Rejected\n";
        file << "EXP_130,EMP_030,Salma_Ibrahim,Travel,1500,2025-01-10,Approved\n";
        file << "EXP_131,EMP_031,Karim_Najib,Food,3200,2025-02-15,Pending\n";
        file << "EXP_132,EMP_032,Nadia_Hassan,Equipment,5000,2025-03-20,Rejected\n";
        file << "EXP_133,EMP_033,Khalid_Ahmed,Utilities,7800,2025-04-05,Approved\n";
        file << "EXP_134,EMP_034,Rihana_Malik,Maintenance,12000,2025-05-12,Pending\n";
        file << "EXP_135,EMP_035,Ibrahim_Samir,Travel,500,2025-06-18,Rejected\n";
        file << "EXP_136,EMP_036,Dina_Farooq,Food,9500,2025-01-10,Approved\n";
        file << "EXP_137,EMP_037,Sarmad_Ali,Equipment,4200,2025-02-15,Pending\n";
        file << "EXP_138,EMP_038,Mona_Yousuf,Utilities,6700,2025-03-20,Rejected\n";
        file << "EXP_139,EMP_039,Nassim_Mohamed,Maintenance,2300,2025-04-05,Approved\n";
        file << "EXP_140,EMP_040,Rania_Abdullah,Travel,8900,2025-05-12,Pending\n";
        file << "EXP_141,EMP_041,Samir_Hassan,Food,11500,2025-06-18,Rejected\n";
        file << "EXP_142,EMP_042,Zeina_Karim,Equipment,3700,2025-01-10,Approved\n";
        file << "EXP_143,EMP_043,Faisal_Rashid,Utilities,15000,2025-02-15,Pending\n";
        file << "EXP_144,EMP_044,Hanan_Ibrahim,Maintenance,800,2025-03-20,Rejected\n";
        file << "EXP_145,EMP_045,Walid_Salim,Travel,1500,2025-04-05,Approved\n";
        file << "EXP_146,EMP_046,Mira_Najib,Food,3200,2025-05-12,Pending\n";
        file << "EXP_147,EMP_047,Yousef_Hassan,Equipment,5000,2025-06-18,Rejected\n";
        file << "EXP_148,EMP_048,Amara_Ahmed,Utilities,7800,2025-01-10,Approved\n";
        file << "EXP_149,EMP_049,Darius_Malik,Maintenance,12000,2025-02-15,Pending\n";
        file << "EXP_150,EMP_050,Nadia_Samir,Travel,500,2025-03-20,Rejected\n";
        file << "EXP_151,EMP_051,Rashid_Farooq,Food,9500,2025-04-05,Approved\n";
        file << "EXP_152,EMP_052,Hana_Ali,Equipment,4200,2025-05-12,Pending\n";
        file << "EXP_153,EMP_053,Kamal_Yousuf,Utilities,6700,2025-06-18,Rejected\n";
        file << "EXP_154,EMP_054,Samira_Mohamed,Maintenance,2300,2025-01-10,Approved\n";
        file << "EXP_155,EMP_055,Tariq_Abdullah,Travel,8900,2025-02-15,Pending\n";
        file << "EXP_156,EMP_056,Leila_Hassan,Food,11500,2025-03-20,Rejected\n";
        file << "EXP_157,EMP_057,Nadir_Ibrahim,Equipment,3700,2025-04-05,Approved\n";
        file << "EXP_158,EMP_058,Rima_Rashid,Utilities,15000,2025-05-12,Pending\n";
        file << "EXP_159,EMP_059,Majid_Karim,Maintenance,800,2025-06-18,Rejected\n";
        file << "EXP_160,EMP_060,Salma_Salim,Travel,1500,2025-01-10,Approved\n";
        file << "EXP_161,EMP_061,Jalal_Najib,Food,3200,2025-02-15,Pending\n";
        file << "EXP_162,EMP_062,Mona_Hassan,Equipment,5000,2025-03-20,Rejected\n";
        file << "EXP_163,EMP_063,Khalid_Ahmed,Utilities,7800,2025-04-05,Approved\n";
        file << "EXP_164,EMP_064,Yasmin_Malik,Maintenance,12000,2025-05-12,Pending\n";
        file << "EXP_165,EMP_065,Faris_Samir,Travel,500,2025-06-18,Rejected\n";
        file << "EXP_166,EMP_066,Dana_Farooq,Food,9500,2025-01-10,Approved\n";
        file << "EXP_167,EMP_067,Sarmad_Ali,Equipment,4200,2025-02-15,Pending\n";
        file << "EXP_168,EMP_068,Leila_Yousuf,Utilities,6700,2025-03-20,Rejected\n";
        file << "EXP_169,EMP_069,Nassim_Mohamed,Maintenance,2300,2025-04-05,Approved\n";
        file << "EXP_170,EMP_070,Rania_Abdullah,Travel,8900,2025-05-12,Pending\n";
        file << "EXP_171,EMP_071,Samir_Hassan,Food,11500,2025-06-18,Rejected\n";
        file << "EXP_172,EMP_072,Zeina_Ibrahim,Equipment,3700,2025-01-10,Approved\n";
        file << "EXP_173,EMP_073,Faisal_Rashid,Utilities,15000,2025-02-15,Pending\n";
        file << "EXP_174,EMP_074,Hanan_Karim,Maintenance,800,2025-03-20,Rejected\n";
        file << "EXP_175,EMP_075,Walid_Salim,Travel,1500,2025-04-05,Approved\n";
        file << "EXP_176,EMP_076,Mira_Najib,Food,3200,2025-05-12,Pending\n";
        file << "EXP_177,EMP_077,Yousef_Hassan,Equipment,5000,2025-06-18,Rejected\n";
        file << "EXP_178,EMP_078,Amara_Ahmed,Utilities,7800,2025-01-10,Approved\n";
        file << "EXP_179,EMP_079,Darius_Malik,Maintenance,12000,2025-02-15,Pending\n";
        file << "EXP_180,EMP_080,Nadia_Samir,Travel,500,2025-03-20,Rejected\n";
        file << "EXP_181,EMP_081,Rashid_Farooq,Food,9500,2025-04-05,Approved\n";
        file << "EXP_182,EMP_082,Hana_Ali,Equipment,4200,2025-05-12,Pending\n";
        file << "EXP_183,EMP_083,Kamal_Yousuf,Utilities,6700,2025-06-18,Rejected\n";
        file << "EXP_184,EMP_084,Samira_Mohamed,Maintenance,2300,2025-01-10,Approved\n";
        file << "EXP_185,EMP_085,Tariq_Abdullah,Travel,8900,2025-02-15,Pending\n";
        file << "EXP_186,EMP_086,Leila_Hassan,Food,11500,2025-03-20,Rejected\n";
        file << "EXP_187,EMP_087,Nadir_Ibrahim,Equipment,3700,2025-04-05,Approved\n";
        file << "EXP_188,EMP_088,Rima_Rashid,Utilities,15000,2025-05-12,Pending\n";
        file << "EXP_189,EMP_089,Majid_Karim,Maintenance,800,2025-06-18,Rejected\n";
        file << "EXP_190,EMP_090,Salma_Salim,Travel,1500,2025-01-10,Approved\n";
        file << "EXP_191,EMP_091,Jalal_Najib,Food,3200,2025-02-15,Pending\n";
        file << "EXP_192,EMP_092,Mona_Hassan,Equipment,5000,2025-03-20,Rejected\n";
        file << "EXP_193,EMP_093,Khalid_Hassan,Utilities,7800,2025-04-05,Approved\n";
        file << "EXP_194,EMP_094,Yasmin_Ibrahim,Maintenance,12000,2025-05-12,Pending\n";
        file << "EXP_195,EMP_095,Faris_Noor,Travel,500,2025-06-18,Rejected\n";
        file << "EXP_196,EMP_096,Dana_Salim,Food,9500,2025-01-10,Approved\n";
        file << "EXP_197,EMP_097,Sarmad_Yousuf,Equipment,4200,2025-02-15,Pending\n";
        file << "EXP_198,EMP_098,Leila_Rashid,Utilities,6700,2025-03-20,Rejected\n";
        file << "EXP_199,EMP_099,Nassim_Malik,Maintenance,2300,2025-04-05,Approved\n";
        file << "EXP_200,EMP_100,Rania_Ahmed,Travel,8900,2025-05-12,Pending\n";

        file.close();

        cout << "\n[OK] Database initialized with 100 sample records!\n\n";
    }

    checkFile.close();
}

// ============================================
// FUNCTION 1: CHECK UNIQUE EXPENSE ID
// ============================================
bool isUnique(string id) {

    ifstream file("expenses.csv");

    if (!file.is_open()) {
        return true;
    }

    string line;

    getline(file, line); // Skip header

    while (getline(file, line)) {

        stringstream ss(line);

        string firstField;

        getline(ss, firstField, ',');

        if (firstField == id) {

            file.close();

            return false;
        }
    }

    file.close();

    return true;
}

// ============================================
// FUNCTION 2: APPEND NEW RECORD
// ============================================
void appendRecord(string data) {

    ofstream file("expenses.csv", ios::app);

    if (file.is_open()) {

        file << data << "\n";

        file.close();

        cout << "\n[OK] Record added successfully!\n";

        stringstream ss(data);

        string token;

        vector<string> fields;

        while (getline(ss, token, ',')) {

            fields.push_back(token);
        }

        if (fields.size() >= 5) {

            string amtStr = fields[4];

            double amount = atof(amtStr.c_str());

            if (amount > 10000) {

                cout << "[WARNING] Amount exceeds Rs.10,000!\n";
                cout << "Manager approval required.\n";
            }
        }
    }
    else {

        cout << "[ERROR] Cannot open file!\n";
    }
}

// ============================================
// FUNCTION 3: SEARCH BY EMPLOYEE ID
// ============================================
void searchByID(string empID) {

    ifstream file("expenses.csv");

    if (!file.is_open()) {

        cout << "[ERROR] Cannot open database!\n";

        return;
    }

    string line;

    bool found = false;

    double totalAmount = 0;

    double approvedAmount = 0;

    double pendingAmount = 0;

    int count = 0;

    // Capture employee name from first matching record
    string empName = "";

    cout << "\n========================================\n";

    cout << " EXPENSE RECORDS FOR: " << empID << "\n";

    cout << "========================================\n";

    cout << "ExpID | Employee Name | Category | Amount | Date | Status\n";

    cout << "----------------------------------------------------------\n";

    getline(file, line); // Skip header

    while (getline(file, line)) {

        stringstream ss(line);

        string token;

        vector<string> fields;

        while (getline(ss, token, ',')) {

            fields.push_back(token);
        }

        if (fields.size() >= 7 && fields[1] == empID) {

            found = true;

            count++;

            // Save employee name from first record found
            if (empName == "") {
                empName = fields[2];
            }

            double amount = atof(fields[4].c_str());

            totalAmount += amount;

            if (fields[6] == "Approved") {

                approvedAmount += amount;
            }
            else if (fields[6] == "Pending") {

                pendingAmount += amount;
            }

            cout << fields[0] << " | "
                 << fields[2] << " | "
                 << fields[3] << " | Rs."
                 << fields[4] << " | "
                 << fields[5] << " | "
                 << fields[6] << "\n";
        }
    }

    file.close();

    if (!found) {

        cout << "\nNo records found for Employee ID: "
             << empID << "\n";
    }
    else {

        cout << "========================================\n";

        cout << "Employee Name   : " << empName << "\n";

        cout << "Employee ID     : " << empID << "\n";

        cout << "----------------------------------------\n";

        cout << "Total Expenses  : Rs." << totalAmount << "\n";

        cout << "Approved Amount : Rs." << approvedAmount << "\n";

        cout << "Pending Amount  : Rs." << pendingAmount << "\n";

        cout << "Total Records   : " << count << "\n";

        cout << "========================================\n";
    }
}

// ============================================
// FUNCTION 4: UPDATE RECORD STATUS
// ============================================
void updateRecord(string expID, string newStatus) {

    ifstream file("expenses.csv");

    ofstream tempFile("temp.csv");

    string line;

    bool updated = false;

    while (getline(file, line)) {

        stringstream ss(line);

        string token;

        vector<string> fields;

        while (getline(ss, token, ',')) {

            fields.push_back(token);
        }

        if (fields.size() >= 7 && fields[0] == expID) {

            fields[6] = newStatus;

            string updatedLine = "";

            for (int i = 0; i < (int)fields.size(); i++) {

                updatedLine += fields[i];

                if (i != (int)fields.size() - 1) {

                    updatedLine += ",";
                }
            }

            tempFile << updatedLine << "\n";

            updated = true;
        }
        else {

            tempFile << line << "\n";
        }
    }

    file.close();

    tempFile.close();

    remove("expenses.csv");

    rename("temp.csv", "expenses.csv");

    if (updated) {

        cout << "\n[OK] Record updated successfully!\n";
    }
    else {

        cout << "\n[ERROR] Record not found!\n";
    }
}

// ============================================
// FUNCTION 5: DISPLAY ALL RECORDS
// ============================================
void displayAllRecords() {

    ifstream file("expenses.csv");

    if (!file.is_open()) {

        cout << "[ERROR] Cannot open database!\n";

        return;
    }

    string line;

    int count = 0;

    cout << "\n========================================\n";

    cout << " ALL EXPENSE RECORDS\n";

    cout << "========================================\n";

    cout << left << setw(10) << "ExpID"
         << setw(12) << "EmpID"
         << setw(15) << "Employee"
         << setw(12) << "Category"
         << setw(10) << "Amount"
         << setw(12) << "Date"
         << setw(10) << "Status\n";

    cout << "========================================\n";

    getline(file, line); // Skip header

    while (getline(file, line)) {

        stringstream ss(line);

        string token;

        vector<string> fields;

        while (getline(ss, token, ',')) {

            fields.push_back(token);
        }

        if (fields.size() >= 7) {

            count++;

            cout << left << setw(10) << fields[0]
                 << setw(12) << fields[1]
                 << setw(15) << fields[2]
                 << setw(12) << fields[3]
                 << setw(10) << "Rs." + fields[4]
                 << setw(12) << fields[5]
                 << setw(10) << fields[6] << "\n";
        }
    }

    file.close();

    cout << "========================================\n";

    cout << "Total Records: " << count << "\n";

    cout << "========================================\n";
}

// ============================================
// FUNCTION 6: SEARCH BY CATEGORY
// ============================================
void searchByCategory(string category) {

    ifstream file("expenses.csv");

    if (!file.is_open()) {

        cout << "[ERROR] Cannot open database!\n";

        return;
    }

    string line;

    bool found = false;

    double totalAmount = 0;

    int count = 0;

    cout << "\n========================================\n";

    cout << " EXPENSES IN CATEGORY: " << category << "\n";

    cout << "========================================\n";

    cout << "ExpID | Employee | Amount | Date | Status\n";

    cout << "------------------------------------------\n";

    getline(file, line); // Skip header

    while (getline(file, line)) {

        stringstream ss(line);

        string token;

        vector<string> fields;

        while (getline(ss, token, ',')) {

            fields.push_back(token);
        }

        if (fields.size() >= 7 && fields[3] == category) {

            found = true;

            count++;

            double amount = atof(fields[4].c_str());

            totalAmount += amount;

            cout << fields[0] << " | "
                 << fields[2] << " | Rs."
                 << fields[4] << " | "
                 << fields[5] << " | "
                 << fields[6] << "\n";
        }
    }

    file.close();

    if (!found) {

        cout << "\nNo records found in category: "
             << category << "\n";
    }
    else {

        cout << "========================================\n";

        cout << "Total Amount    : Rs." << totalAmount << "\n";

        cout << "Total Records   : " << count << "\n";

        cout << "========================================\n";
    }
}

// ============================================
// FUNCTION 7: SEARCH BY STATUS
// ============================================
void searchByStatus(string status) {

    ifstream file("expenses.csv");

    if (!file.is_open()) {

        cout << "[ERROR] Cannot open database!\n";

        return;
    }

    string line;

    bool found = false;

    double totalAmount = 0;

    int count = 0;

    cout << "\n========================================\n";

    cout << " EXPENSES WITH STATUS: " << status << "\n";

    cout << "========================================\n";

    cout << "ExpID | Employee | Category | Amount | Date\n";

    cout << "------------------------------------------\n";

    getline(file, line); // Skip header

    while (getline(file, line)) {

        stringstream ss(line);

        string token;

        vector<string> fields;

        while (getline(ss, token, ',')) {

            fields.push_back(token);
        }

        if (fields.size() >= 7 && fields[6] == status) {

            found = true;

            count++;

            double amount = atof(fields[4].c_str());

            totalAmount += amount;

            cout << fields[0] << " | "
                 << fields[2] << " | "
                 << fields[3] << " | Rs."
                 << fields[4] << " | "
                 << fields[5] << "\n";
        }
    }

    file.close();

    if (!found) {

        cout << "\nNo records found with status: "
             << status << "\n";
    }
    else {

        cout << "========================================\n";

        cout << "Total Amount    : Rs." << totalAmount << "\n";

        cout << "Total Records   : " << count << "\n";

        cout << "========================================\n";
    }
}

// ============================================
// FUNCTION 8: DELETE RECORD
// ============================================
void deleteRecord(string expID) {

    ifstream file("expenses.csv");

    if (!file) {

        cout << "\n[ERROR] Cannot open file!\n";
        return;
    }

    vector<string> allLines;

    string line;

    bool found = false;

    getline(file, line);

    allLines.push_back(line);

    while (getline(file, line)) {

        stringstream ss(line);

        string token;

        vector<string> fields;

        while (getline(ss, token, ',')) {

            fields.push_back(token);
        }

        if (fields.size() >= 1 && fields[0] == expID) {

            found = true;
            continue;
        }

        allLines.push_back(line);
    }

    file.close();

    ofstream outFile("expenses.csv");

    for (int i = 0; i < (int)allLines.size(); i++) {

        outFile << allLines[i] << endl;
    }

    outFile.close();

    if (found) {

        cout << "\n[OK] Record deleted successfully!\n";
    }
    else {

        cout << "\n[ERROR] Expense ID not found!\n";
    }
}

// ============================================
// FUNCTION 9: DISPLAY STATISTICS
// ============================================
void displayStatistics() {

    ifstream file("expenses.csv");

    if (!file.is_open()) {

        cout << "[ERROR] Cannot open database!\n";

        return;
    }

    string line;

    double totalExpenses = 0;

    double approvedTotal = 0;

    double rejectedTotal = 0;

    double pendingTotal = 0;

    int approvedCount = 0;

    int rejectedCount = 0;

    int pendingCount = 0;

    getline(file, line); // Skip header

    while (getline(file, line)) {

        stringstream ss(line);

        string token;

        vector<string> fields;

        while (getline(ss, token, ',')) {

            fields.push_back(token);
        }

        if (fields.size() >= 7) {

            double amount = atof(fields[4].c_str());

            totalExpenses += amount;

            if (fields[6] == "Approved") {

                approvedTotal += amount;

                approvedCount++;
            }
            else if (fields[6] == "Rejected") {

                rejectedTotal += amount;

                rejectedCount++;
            }
            else if (fields[6] == "Pending") {

                pendingTotal += amount;

                pendingCount++;
            }
        }
    }

    file.close();

    cout << "\n========================================\n";

    cout << "      EXPENSE STATISTICS\n";

    cout << "========================================\n";

    cout << "Total Expenses   : Rs." << totalExpenses << "\n";

    cout << "Approved Amount  : Rs." << approvedTotal << " (" << approvedCount << " records)\n";

    cout << "Rejected Amount  : Rs." << rejectedTotal << " (" << rejectedCount << " records)\n";

    cout << "Pending Amount   : Rs." << pendingTotal << " (" << pendingCount << " records)\n";

    cout << "========================================\n";
}

// ============================================
// FUNCTION 10: DISPLAY MENU
// ============================================
void showMenu() {

    cout << "\n========================================\n";

    cout << " CORPORATE EXPENSE TRACKER SYSTEM\n";

    cout << "========================================\n";

    cout << "1. Add New Expense\n";

    cout << "2. Search by Employee ID\n";

    cout << "3. Update Expense Status\n";

    cout << "4. Display All Records\n";

    cout << "5. Search by Category\n";

    cout << "6. Search by Status\n";

    cout << "7. Delete Expense Record\n";

    cout << "8. View Statistics\n";

    cout << "9. Exit\n";

    cout << "========================================\n";

    cout << "Enter Choice: ";
}

// ============================================
// MAIN FUNCTION
// ============================================
int main() {

    // ============================================
    // STEP 1: ESP32 HARDWARE AUTHENTICATION
    // Program will NOT continue without valid key
    // ============================================
    if (!authenticateESP32()) {
        cout << "\n========================================\n";
        cout << " ACCESS DENIED - PROGRAM TERMINATED\n";
        cout << "========================================\n";
        cout << "Press Enter to exit...";
        cin.get();
        return 1;  // Exit with error code
    }

    // ============================================
    // STEP 2: INITIALIZE DATABASE (only if auth OK)
    // ============================================
    initializeDatabase();

    // ============================================
    // STEP 3: RUN MAIN PROGRAM LOOP
    // ============================================
    int choice;

    while (true) {

        showMenu();

        cin >> choice;

        cin.ignore();

        if (choice == 1) {

            string expID;
            string empID;
            string empName;
            string category;
            string amount;
            string date;

            cout << "\n--- ADD NEW EXPENSE ---\n";

            cout << "Expense ID: ";
            getline(cin, expID);

            if (!isUnique(expID)) {

                cout << "\n[ERROR] Expense ID already exists!\n";

                continue;
            }

            cout << "Employee ID: ";
            getline(cin, empID);

            cout << "Employee Name: ";
            getline(cin, empName);

            cout << "Category (Food/Equipment/Utilities/Maintenance/Travel): ";
            getline(cin, category);

            cout << "Amount: ";
            getline(cin, amount);

            cout << "Date (YYYY-MM-DD): ";
            getline(cin, date);

            string status = "Pending";

            string record =
                expID + "," +
                empID + "," +
                empName + "," +
                category + "," +
                amount + "," +
                date + "," +
                status;

            appendRecord(record);
        }

        else if (choice == 2) {

            string empID;

            cout << "\nEnter Employee ID: ";

            getline(cin, empID);

            searchByID(empID);
        }

        else if (choice == 3) {

            string expID;

            string newStatus;

            cout << "\nEnter Expense ID: ";

            getline(cin, expID);

            cout << "Enter New Status (Approved/Rejected/Pending): ";

            getline(cin, newStatus);

            updateRecord(expID, newStatus);
        }

        else if (choice == 4) {

            displayAllRecords();
        }

        else if (choice == 5) {

            string category;

            cout << "\nEnter Category (Food/Equipment/Utilities/Maintenance/Travel): ";

            getline(cin, category);

            searchByCategory(category);
        }

        else if (choice == 6) {

            string status;

            cout << "\nEnter Status (Approved/Rejected/Pending): ";

            getline(cin, status);

            searchByStatus(status);
        }

        else if (choice == 7) {

            string expID;

            cout << "\nEnter Expense ID to delete: ";

            getline(cin, expID);

            deleteRecord(expID);
        }

        else if (choice == 8) {

            displayStatistics();
        }

        else if (choice == 9) {

            cout << "\nThank you for using Expense Tracker!\n";

            cout << "Goodbye!\n\n";

            break;
        }

        else {

            cout << "\n[ERROR] Invalid choice! Please try again.\n";
        }
    }

    return 0;
}
