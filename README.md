# Final-Project-Milestone
#include <iostream>
#include <fstream>
#include <sstream>
#include <unordered_map>
#include <deque>
#include <string>
#include <algorithm>

using namespace std;

// Cache definitions
unordered_map<string, string> cache;
deque<string> cacheOrder;
const int CACHE_SIZE = 10;          // Maximum cache size

// Helper: Convert to lowercase for matching
string toLower(const string& str) {
    string lowerStr = str;
    transform(lowerStr.begin(), lowerStr.end(), lowerStr.begin(), ::tolower);
    return lowerStr;
}

// Lookup in cache
bool lookupCache(const string& key, string& population) {
    auto it = cache.find(key);      // Search for the key in cache
    if (it != cache.end()) {
        population = it->second;    // Retrieve population if found
    return true;                    // Found in cache
    }
    return false;                   // Not found in cache
}

// Add to cache with eviction logic
void addToCache(const string& key, const string& population) {
    // Only add if not already present
    if (cache.find(key) == cache.end()) {
        // If cache is full, remove the oldest entry (FIFO logic)
        if (cacheOrder.size() == CACHE_SIZE) {
            string oldest = cacheOrder.front();
            cacheOrder.pop_front();   // Remove from order tracking
            cache.erase(oldest);      // Remove from actual cache
        }
        cacheOrder.push_back(key);    // Add new entry to order tracking
        cache[key] = population;      // Add new entry to cache
    }
}

// CSV file search
bool searchCSV(const string& filename, const string& country, const string& city, string& population) {
    ifstream file(filename);            // Open the file
    if (!file.is_open()) {          // Check if the file was opened successfully
        cerr << "Error opening file.\n";
        return false;
    }

    string line, csvCountry, csvCity, csvPopulation;
    getline(file, line); // Skip header

    while (getline(file, line)) {
        stringstream ss(line);
        getline(ss, csvCountry, ',');       // Read county code
        getline(ss, csvCity, ',');          // Read city name
        getline(ss, csvPopulation, ',');    // Read Population

        if (toLower(csvCountry) == toLower(country) && toLower(csvCity) == toLower(city)) {
            population = csvPopulation;
            file.close();       // Close the file after finding the match
            return true;
        }
    }

    file.close();       // Close the file if no match is found
    return false;
}

int main() {
    string country, city, key, population;
    const string filename = "world_cities.csv";

    while (true) {
        cout << "\nEnter city name (or 'exit' to quit): ";
        getline(cin, city);
        if (toLower(city) == "exit") break;

        cout << "Enter country code: ";
        getline(cin, country);

        key = toLower(country) + "," + toLower(city);

        if (lookupCache(key, population)) {
            cout << "Population (from cache): " << population << endl;
        } else {
            if (searchCSV(filename, country, city, population)) {
                cout << "Population (from file): " << population << endl;
                addToCache(key, population);
            } else {
                cout << "City not found.\n";
            }
        }

        // Debug cache
        cout << "Current Cache:\n";
        for (const auto& k : cacheOrder) {
            cout << k << " -> " << cache[k] << endl;
        }
    }

    return 0;
}
