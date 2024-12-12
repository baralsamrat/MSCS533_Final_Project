# MSCS533_Final_Project


using Microsoft.Maui.Controls;
using Microsoft.Maui.Essentials;
using Microsoft.Maui.Maps;
using SQLite;
using System;
using System.Collections.ObjectModel;
using System.Linq;
using System.Timers;

namespace LocationHeatmapApp
{
    public partial class MainPage : ContentPage
    {
        private Timer locationTimer; // Timer for periodic location tracking
        private SQLiteConnection db; // SQLite database connection
        private ObservableCollection<LocationData> locations; // Observable list of locations for UI binding

        public MainPage()
        {
            Console.WriteLine("Initializing MainPage...");
            InitializeComponent();

            // Initialize SQLite
            string dbPath = System.IO.Path.Combine(Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData), "locations.db");
            Console.WriteLine($"Database path: {dbPath}");
            db = new SQLiteConnection(dbPath);
            db.CreateTable<LocationData>(); // Create table if it doesn't exist

            // Load existing locations from the database into the observable collection
            locations = new ObservableCollection<LocationData>(db.Table<LocationData>().ToList());
            Console.WriteLine("Loaded locations from database.");
            LocationMap.ItemsSource = locations; // Bind locations to the map

            // Set up the timer to track location every 5 seconds
            locationTimer = new Timer(5000);
            locationTimer.Elapsed += TrackLocation;
            locationTimer.AutoReset = true; // Ensure the timer repeats
            Console.WriteLine("Location tracking timer initialized.");
        }

        private async void TrackLocation(object sender, ElapsedEventArgs e)
        {
            Console.WriteLine("Tracking location...");
            try
            {
                // Attempt to get the last known location; fall back to fetching the current location
                var location = await Geolocation.GetLastKnownLocationAsync() ?? await Geolocation.GetLocationAsync();

                if (location != null)
                {
                    Console.WriteLine($"Location acquired: Latitude = {location.Latitude}, Longitude = {location.Longitude}");

                    // Create a new location data entry
                    var locationData = new LocationData { Latitude = location.Latitude, Longitude = location.Longitude, Timestamp = DateTime.Now };

                    // Insert the location data into the database
                    lock (db) // Ensure thread safety for database operations
                    {
                        db.Insert(locationData);
                        Console.WriteLine("Location data inserted into database.");
                    }

                    // Update the UI on the main thread
                    MainThread.BeginInvokeOnMainThread(() => {
                        locations.Add(locationData);
                        Console.WriteLine("Location data added to ObservableCollection.");
                    });
                }
                else
                {
                    Console.WriteLine("No location data available.");
                }
            }
            catch (Exception ex)
            {
                // Handle and log exceptions during location tracking
                Console.WriteLine($"Location tracking error: {ex.Message}");
            }
        }

        private void StartTracking_Clicked(object sender, EventArgs e)
        {
            // Start the location tracking timer
            Console.WriteLine("Start tracking button clicked.");
            locationTimer.Start();
            Console.WriteLine("Location tracking started.");
        }

        private void StopTracking_Clicked(object sender, EventArgs e)
        {
            // Stop the location tracking timer
            Console.WriteLine("Stop tracking button clicked.");
            locationTimer.Stop();
            Console.WriteLine("Location tracking stopped.");
        }

        protected override void OnDisappearing()
        {
            base.OnDisappearing();
            // Dispose of the timer to free resources
            locationTimer.Stop();
            locationTimer.Dispose();
            Console.WriteLine("Timer disposed.");
        }
    }

    public class LocationData
    {
        [PrimaryKey, AutoIncrement]
        public int Id { get; set; } // Unique ID for each location entry

        public double Latitude { get; set; } // Latitude of the location
        public double Longitude { get; set; } // Longitude of the location
        public DateTime Timestamp { get; set; } // Timestamp of the location entry
    }

    public class HeatMap : Map
    {
        public ObservableCollection<LocationData> HeatPoints { get; set; } // Data source for heat points

        public HeatMap()
        {
            Console.WriteLine("Initializing HeatMap...");
            HeatPoints = new ObservableCollection<LocationData>();
            this.PropertyChanged += (s, e) => UpdateHeatMap(); // Update the map when properties change
            Console.WriteLine("HeatMap initialized.");
        }

        private void UpdateHeatMap()
        {
            Console.WriteLine("Updating HeatMap...");
            this.MapElements.Clear(); // Prevent duplicate elements

            foreach (var point in HeatPoints)
            {
                Console.WriteLine($"Adding heat point: Latitude = {point.Latitude}, Longitude = {point.Longitude}");

                // Add a visual representation of the heat point
                var circle = new Circle
                {
                    Center = new Location(point.Latitude, point.Longitude),
                    Radius = new Distance(50), // Adjust for intensity
                    StrokeColor = Color.Transparent,
                    FillColor = Color.FromRgba(255, 0, 0, 50) // Semi-transparent red
                };

                this.MapElements.Add(circle);
                Console.WriteLine("Heat point added to map.");
            }
        }
    }

    public partial class AppShell : Shell
    {
        public AppShell()
        {
            Console.WriteLine("Initializing AppShell...");
            // Register the main page route
            Routing.RegisterRoute(nameof(MainPage), typeof(MainPage));
            Console.WriteLine("AppShell initialized.");
        }
    }

    public partial class App : Application
    {
        public App()
        {
            Console.WriteLine("Initializing App...");
            InitializeComponent();
            MainPage = new AppShell(); // Set the main page to the AppShell
            Console.WriteLine("App initialized.");
        }
    }
}
