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

// FinalProjectMilestone2
#include <iostream>
#include <fstream>
#include <sstream>
#include <unordered_map>
#include <queue>
#include <list>
#include <vector>
#include <random>
#include <memory>
#include <algorithm>

// Helper function: convert string to lowercase for consistent key matching
std::string toLower(const std::string& str) {
    std::string lowerStr = str;
    std::transform(lowerStr.begin(), lowerStr.end(), lowerStr.begin(), ::tolower);
    return lowerStr;
}

// Base interface for all caching strategies
class ICacheStrategy {
public:
    // Insert a key and its population value into the cache
    virtual void insert(const std::string& key, double population) = 0;

    // Try to get the population for a key; return true if found in cache
    virtual bool get(const std::string& key, double& population) = 0;

    // Display the current contents of the cache
    virtual void displayCache() const = 0;

    virtual ~ICacheStrategy() {}
};

// LFU (Least Frequently Used) cache implementation
class LFUCache : public ICacheStrategy {
    struct Entry {
        double population;  // Stored population value
        int freq;           // Frequency of access
        int order;          // Order of insertion for tie-breaking
    };

    std::unordered_map<std::string, Entry> cache;
    int currentOrder = 0;
    const size_t capacity = 10;

public:
    // Insert a new entry or update existing one, evict LFU if needed
    void insert(const std::string& key, double population) override {
        currentOrder++;
        if (cache.count(key)) {
            cache[key].population = population;
            cache[key].freq++;
            return;
        }
        if (cache.size() >= capacity) {
            // Find LFU entry to evict (oldest if frequency is tied)
            auto evictKey = std::min_element(cache.begin(), cache.end(), [](const auto& a, const auto& b) {
                if (a.second.freq == b.second.freq)
                    return a.second.order < b.second.order;
                return a.second.freq < b.second.freq;
            })->first;
            cache.erase(evictKey);
        }
        cache[key] = { population, 1, currentOrder };
    }

    // Retrieve from cache and increment frequency counter
    bool get(const std::string& key, double& population) override {
        if (cache.count(key)) {
            population = cache[key].population;
            cache[key].freq++;
            return true;
        }
        return false;
    }

    // Print LFU cache content and frequencies
    void displayCache() const override {
        std::cout << "[LFU Cache]\n";
        for (const auto& [k, v] : cache)
            std::cout << k << " => " << v.population << " (freq=" << v.freq << ")\n";
    }
};

// FIFO (First-In, First-Out) cache implementation
class FIFOCache : public ICacheStrategy {
    std::unordered_map<std::string, double> cache;
    std::queue<std::string> order;
    const size_t capacity = 10;

public:
    // Insert new entry, evict the oldest if cache is full
    void insert(const std::string& key, double population) override {
        if (cache.count(key)) return;
        if (cache.size() >= capacity) {
            std::string oldest = order.front(); order.pop();
            cache.erase(oldest);
        }
        cache[key] = population;
        order.push(key);
    }

    // Lookup population in FIFO cache
    bool get(const std::string& key, double& population) override {
        if (cache.count(key)) {
            population = cache[key];
            return true;
        }
        return false;
    }

    // Print FIFO cache content
    void displayCache() const override {
        std::cout << "[FIFO Cache]\n";
        for (const auto& [k, v] : cache)
            std::cout << k << " => " << v << "\n";
    }
};

// Random Replacement cache implementation
class RandomCache : public ICacheStrategy {
    std::unordered_map<std::string, double> cache;
    std::vector<std::string> keys;
    std::default_random_engine gen{ std::random_device{}() };
    const size_t capacity = 10;

public:
    // Insert new entry, evict a random one if full
    void insert(const std::string& key, double population) override {
        if (cache.count(key)) return;
        if (cache.size() >= capacity) {
            std::uniform_int_distribution<> dist(0, keys.size() - 1);
            std::string evictKey = keys[dist(gen)];
            cache.erase(evictKey);
            keys.erase(std::remove(keys.begin(), keys.end(), evictKey), keys.end());
        }
        cache[key] = population;
        keys.push_back(key);
    }

    // Lookup in random cache
    bool get(const std::string& key, double& population) override {
        if (cache.count(key)) {
            population = cache[key];
            return true;
        }
        return false;
    }

    // Print contents of random cache
    void displayCache() const override {
        std::cout << "[Random Cache]\n";
        for (const auto& [k, v] : cache)
            std::cout << k << " => " << v << "\n";
    }
};

// Function to search for a city's population in the CSV file
bool searchCSV(const std::string& filename, const std::string& country, const std::string& city, std::string& population) {
    std::ifstream file(filename);
    if (!file.is_open()) {
        std::cerr << "Error opening file.\n";
        return false;
    }

    std::string line, csvCountry, csvCity, csvPopulation;
    getline(file, line); // Skip header

    while (getline(file, line)) {
        std::stringstream ss(line);
        getline(ss, csvCountry, ',');
        getline(ss, csvCity, ',');
        getline(ss, csvPopulation, ',');

        // Match country and city in a case-insensitive manner
        if (toLower(csvCountry) == toLower(country) && toLower(csvCity) == toLower(city)) {
            population = csvPopulation;
            file.close();
            return true;
        }
    }

    file.close();
    return false;
}

// Main driver function
int main() {
    std::unique_ptr<ICacheStrategy> cache; // Pointer to selected caching strategy
    int choice;
    std::string country, city, key, population;
    const std::string filename = "world_cities.csv"; // Input CSV file

    // Prompt user to select caching strategy
    std::cout << "Choose cache strategy: 1 = LFU, 2 = FIFO, 3 = Random: ";
    std::cin >> choice;
    std::cin.ignore(); // Clear input buffer

    // Initialize selected cache implementation
    if (choice == 1)
        cache = std::make_unique<LFUCache>();
    else if (choice == 2)
        cache = std::make_unique<FIFOCache>();
    else
        cache = std::make_unique<RandomCache>();

    // Main interaction loop
    while (true) {
        std::cout << "\nEnter city name (or 'exit' to quit): ";
        getline(std::cin, city);
        if (toLower(city) == "exit") break;

        std::cout << "Enter country code: ";
        getline(std::cin, country);

        // Create a lowercase key for consistent cache lookup
        key = toLower(country) + "," + toLower(city);
        double popVal;

        // First check if value is in cache
        if (cache->get(key, popVal)) {
            std::cout << "Population (from cache): " << popVal << std::endl;
        } else {
            // If not in cache, search the CSV file
            if (searchCSV(filename, country, city, population)) {
                std::cout << "Population (from file): " << population << std::endl;
                cache->insert(key, std::stod(population));
            } else {
                std::cout << "City not found.\n";
            }
        }

        // Display the current state of the cache
        cache->displayCache();
    }

    return 0;
}
