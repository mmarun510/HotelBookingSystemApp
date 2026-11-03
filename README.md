# HotelBookingSystemApp

import java.sql.*;
import java.time.LocalDate;
import java.time.temporal.ChronoUnit;
import java.util.Scanner;

public class HotelBookingSystemApp {
    private static final String MYSQL_SERVER_URL = "jdbc:mysql://localhost:3306/";
    private static final String DB_NAME = "hotel_db";
    private static final String DB_URL = MYSQL_SERVER_URL + DB_NAME;
    private static final String DB_USER = "root"; 
    private static final String DB_PASSWORD = "password"; 

    public static void main(String[] args) {
        initializeDatabaseSchema();

        Scanner scanner = new Scanner(System.in);

        try (Connection conn = DriverManager.getConnection(DB_URL, DB_USER, DB_PASSWORD)) {
            // Enable manual transaction control for operations affecting multiple tables
            conn.setAutoCommit(false); 

            boolean exit = false;
            while (!exit) {
                System.out.println("\n===== HOTEL ROOM BOOKING SYSTEM =====");
                System.out.println("1. View Room Availability");
                System.out.println("2. Make a Room Booking");
                System.out.println("3. Check-In");
                System.out.println("4. Check-Out & Generate Bill");
                System.out.println("5. Exit");
                System.out.print("Choose an option: ");

                int choice = scanner.nextInt();
                scanner.nextLine(); // Consume newline character

                switch (choice) {
                    case 1:
                        viewRoomAvailability(conn);
                        break;
                    case 2:
                        makeBooking(conn, scanner);
                        break;
                    case 3:
                        checkIn(conn, scanner);
                        break;
                    case 4:
                        checkOutAndBill(conn, scanner);
                        break;
                    case 5:
                        exit = true;
                        System.out.println("Exiting system. Goodbye!");
                        break;
                    default:
                        System.out.println("Invalid choice. Try again.");
                }
            }
        } catch (SQLException e) {
            System.err.println("Database execution error: " + e.getMessage());
            e.printStackTrace();
        } finally {
            scanner.close();
        }
    }
    private static void initializeDatabaseSchema() {
        System.out.println("Connecting to MySQL server and establishing database configurations...");
        
        try {
            // Register modern MySQL driver explicitly to ensure compatibility
            Class.forName("com.mysql.cj.jdbc.Driver");
        } catch (ClassNotFoundException e) {
            System.err.println("MySQL Connector/J Driver not found. Please add the dependency jar file to your classpath.");
            return;
        }

        try (Connection rootConn = DriverManager.getConnection(MYSQL_SERVER_URL, DB_USER, DB_PASSWORD);
             Statement stmt = rootConn.createStatement()) {
            
            stmt.executeUpdate("CREATE DATABASE IF NOT EXISTS " + DB_NAME);
            System.out.println("Database '" + DB_NAME + "' verified/created successfully.");
            
        } catch (SQLException e) {
            System.err.println("Initial server-level schema building failed: " + e.getMessage());
            return;
        }

        // Establish connection to targeted application database to provision schema
        try (Connection appConn = DriverManager.getConnection(DB_URL, DB_USER, DB_PASSWORD);
             Statement stmt = appConn.createStatement()) {

            // Create Rooms Architecture Table
            stmt.executeUpdate("CREATE TABLE IF NOT EXISTS rooms (" +
                    "room_id INT PRIMARY KEY AUTO_INCREMENT, " +
                    "room_type VARCHAR(50) NOT NULL, " +
                    "price_per_day DECIMAL(10, 2) NOT NULL, " +
                    "is_available BOOLEAN DEFAULT TRUE)");

            // Create Customers Registry Table
            stmt.executeUpdate("CREATE TABLE IF NOT EXISTS customers (" +
                    "customer_id INT PRIMARY KEY AUTO_INCREMENT, " +
                    "name VARCHAR(100) NOT NULL, " +
                    "phone VARCHAR(20) NOT NULL)");

            // Create Bookings Transaction Table
            stmt.executeUpdate("CREATE TABLE IF NOT EXISTS bookings (" +
                    "booking_id INT PRIMARY KEY AUTO_INCREMENT, " +
                    "customer_id INT, " +
                    "room_id INT, " +
                    "check_in DATE NOT NULL, " +
                    "check_out DATE NOT NULL, " +
                    "status VARCHAR(20) DEFAULT 'BOOKED', " + // BOOKED, CHECKED_IN, CHECKED_OUT
                    "total_amount DECIMAL(10, 2) DEFAULT 0.00, " +
                    "FOREIGN KEY (customer_id) REFERENCES customers(customer_id), " +
                    "FOREIGN KEY (room_id) REFERENCES rooms(room_id))");

            ResultSet checkRooms = stmt.executeQuery("SELECT COUNT(*) FROM rooms");
            if (checkRooms.next() && checkRooms.getInt(1) == 0) {
                stmt.executeUpdate("INSERT INTO rooms (room_type, price_per_day, is_available) VALUES " +
                        "('Single Standard', 1500.00, TRUE), " +
                        "('Executive Double', 2500.00, TRUE), " +
                        "('Premium Deluxe', 4000.00, TRUE), " +
                        "('Luxury Suite', 7500.00, TRUE)");
                System.out.println("Initial lodging portfolio populated with baseline room records.");
            }
            
            System.out.println("Schema layout checks finalized. Starting system runtime environments...");
            
        } catch (SQLException e) {
            System.err.println("Fatal error executing database build commands: " + e.getMessage());
        }
    }

    //  Room Availability
    private static void viewRoomAvailability(Connection conn) throws SQLException {
        String sql = "SELECT room_id, room_type, price_per_day, is_available FROM rooms";
        try (Statement stmt = conn.createStatement();
             ResultSet rs = stmt.executeQuery(sql)) {
            System.out.println("\n--- Current Inventory Status ---");
            System.out.printf("%-10s %-20s %-15s %-15s%n", "Room ID", "Room Type", "Price/Day (₹)", "Availability");
            System.out.println("--------------------------------------------------------------");
            while (rs.next()) {
                System.out.printf("%-10d %-20s %-15.2f %-15s%n",
                        rs.getInt("room_id"),
                        rs.getString("room_type"),
                        rs.getDouble("price_per_day"),
                        rs.getBoolean("is_available") ? "AVAILABLE" : "OCCUPIED / BOOKED");
            }
        }
    }

    //  Booking 
    private static void makeBooking(Connection conn, Scanner scanner) {
        try {
            System.out.print("\nEnter Customer Full Name: ");
            String name = scanner.nextLine();
            System.out.print("Enter Contact Number: ");
            String phone = scanner.nextLine();
            System.out.print("Enter Targeted Room ID: ");
            int roomId = scanner.nextInt();
            scanner.nextLine();
            System.out.print("Enter Check-In Target Date (YYYY-MM-DD): ");
            LocalDate checkInDate = LocalDate.parse(scanner.nextLine());
            System.out.print("Enter Check-Out Target Date (YYYY-MM-DD): ");
            LocalDate checkOutDate = LocalDate.parse(scanner.nextLine());

            if (!checkOutDate.isAfter(checkInDate)) {
                System.out.println("Operational failure: Check-out schedule must succeed check-in date timeline.");
                conn.rollback();
                return;
            }

            // Verify room 
            String checkRoomSql = "SELECT is_available, price_per_day FROM rooms WHERE room_id = ?";
            try (PreparedStatement pstCheck = conn.prepareStatement(checkRoomSql)) {
                pstCheck.setInt(1, roomId);
                ResultSet rs = pstCheck.executeQuery();
                
                if (!rs.next() || !rs.getBoolean("is_available")) {
                    System.out.println("Target room record is currently unavailable or does not exist.");
                    conn.rollback();
                    return;
                }
                
                double pricePerDay = rs.getDouble("price_per_day");
                long lengthOfStay = ChronoUnit.DAYS.between(checkInDate, checkOutDate);
                double dynamicBillEstimate = lengthOfStay * pricePerDay;

                // Register Customer entry
                String insertCustSql = "INSERT INTO customers (name, phone) VALUES (?, ?)";
                int customerId = -1;
                try (PreparedStatement pstCust = conn.prepareStatement(insertCustSql, Statement.RETURN_GENERATED_KEYS)) {
                    pstCust.setString(1, name);
                    pstCust.setString(2, phone);
                    pstCust.executeUpdate();
                    
                    ResultSet generatedKeys = pstCust.getGeneratedKeys();
                    if (generatedKeys.next()) {
                        customerId = generatedKeys.getInt(1);
                    } else {throw new SQLException("Database sequence exception processing user identification keys.");
}
}
//  booking transaction
String insertBookSql = "INSERT INTO bookings (customer_id, room_id, check_in, check_out, status, total_amount) VALUES (?, ?, ?, ?, 'BOOKED', ?)";
try (PreparedStatement pstBook = conn.prepareStatement(insertBookSql)) {
pstBook.setInt(1, customerId);
pstBook.setInt(2, roomId);
pstBook.setDate(3, Date.valueOf(checkInDate));
pstBook.setDate(4, Date.valueOf(checkOutDate));
pstBook.setDouble(5, dynamicBillEstimate);
pstBook.executeUpdate();
}

//  Lock target room status 
String updateRoomSql = "UPDATE rooms SET is_available = FALSE WHERE room_id = ?";
try (PreparedStatement pstUpRoom = conn.prepareStatement(updateRoomSql)) {
pstUpRoom.setInt(1, roomId);
pstUpRoom.executeUpdate();
}
// Atomically finalize workflow
conn.commit();
System.out.println("Reservation locked successfully! Base Aggregate Calculation: ₹" + dynamicBillEstimate);
} catch (SQLException e) {
conn.rollback(); // Rollback transaction on failure
System.err.println("Booking pipeline interrupted. Rolling back mutations: " + e.getMessage());
}
} catch (Exception e) {
System.out.println("Input execution parsing anomaly tracking. Operation suspended.");
}
}
// check-In Module
private static void checkIn(Connection conn, Scanner scanner) {
try {
System.out.print("\nEnter Booking ID Reference to initiate Guest Check-In: ");
int bookingId = scanner.nextInt();
scanner.nextLine();
String sql = "UPDATE bookings SET status = 'CHECKED_IN' WHERE booking_id = ? AND status = 'BOOKED'";
try (PreparedStatement pst = conn.prepareStatement(sql)) {
pst.setInt(1, bookingId);
int rowsUpdated = pst.executeUpdate();
if (rowsUpdated > 0) {
conn.commit();
System.out.println("Guest check-in workflow finalized for Reservation ID: " + bookingId);
} else {
System.out.println("Active reservation pointer not matched or guest is already checked-in.");
conn.rollback();
}
}
} catch (SQLException e) {
try { conn.rollback(); } catch (SQLException ex) { ex.printStackTrace(); }
System.err.println("Error transitioning system workflow to Checked-In: " + e.getMessage());
}
}
// check-Out & Billing Module
private static void checkOutAndBill(Connection conn, Scanner scanner) {
try {
System.out.print("\nEnter active Booking ID Reference to initiate Check-Out: ");
int bookingId = scanner.nextInt();
scanner.nextLine();
String fetchSql = "SELECT b.booking_id, b.room_id, b.total_amount, c.name " +
"FROM bookings b JOIN customers c ON b.customer_id = c.customer_id " +
"WHERE b.booking_id = ? AND b.status IN ('BOOKED', 'CHECKED_IN')";
try (PreparedStatement pstFetch = conn.prepareStatement(fetchSql)) {
pstFetch.setInt(1, bookingId);
ResultSet rs = pstFetch.executeQuery();
if (rs.next()) {
int roomId = rs.getInt("room_id");
double finalSettlementDue = rs.getDouble("total_amount");
String customerLabel = rs.getString("name");

//  Modify ledger
String updateBookSql = "UPDATE bookings SET status = 'CHECKED_OUT' WHERE booking_id = ?";
try (PreparedStatement pstUpBook = conn.prepareStatement(updateBookSql)) {
pstUpBook.setInt(1, bookingId);
pstUpBook.executeUpdate();
}
//  Release structural room locks back to available pools
String updateRoomSql = "UPDATE rooms SET is_available = TRUE WHERE room_id = ?";
try (PreparedStatement pstUpRoom = conn.prepareStatement(updateRoomSql)) {
pstUpRoom.setInt(1, roomId);
pstUpRoom.executeUpdate();
}
conn.commit();
// Invoice Billing 
System.out.println("\n==================================");
System.out.println("        INVOICE RECEIPT STATEMENT ");
System.out.println("==================================");
System.out.println("Booking Reference ID : " + bookingId);
System.out.println("Registered Guest Full Name : " + customerLabel);
System.out.println("Room Resource Identifier    : Room " + roomId);
System.out.println("Total Due Settlement         : ₹" + finalSettlementDue);
System.out.println("Status Indicator             : SETTLED & CHECKED_OUT");
System.out.println("==================================");
} else {
System.out.println("No transactional workflow instance matching ID reference: " + bookingId);
conn.rollback();
}
}
} catch (SQLException e) {
try { conn.rollback(); } catch (SQLException ex) { ex.printStackTrace(); }
System.err.println("Fatal accounting pipeline calculation exception: " + e.getMessage());
}
}
}
