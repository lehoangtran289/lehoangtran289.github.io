---
title: "Code refactoring practices"
layout: single
date: 2023-11-05
permalink: /posts/code-refactoring-practices/
author_profile: false
tags:
  - Refactoring
  - Software Development
  - Best Practices
---
Some code refactoring practices that I have thought would be useful in daily software development.

{% include toc %}

## 1. Creation problem

### 1.1. Problem

- Too many constructors

```java
class Vehicle {
    private String make;
    private String model;
    private int year;
    private String plateNumber;

    public Vehicle(String make, String model) {
        this.make = make;
        this.model = model;
    }

    public Vehicle(String make, String model, String plateNumber) {
        this.make = make;
        this.model = model;
        this.plateNumber = plateNumber;
    }
}

public class Main {
    public static void main(String[] args) {
        System.out.println("Hello world!");
        Vehicle unidentifiedCar = new Vehicle("Ferrari", "Scuderia");
        Vehicle identifiedCar = new Vehicle("Ferrari", "Scuderia", "H12-12345");
    }
}
```

### 1.2. Refactoring method

- Use Builder pattern

```java
class Vehicle {
    private String make;
    private String model;
    private int year;
    private String plateNumber;

    private Vehicle(VehicleBuilder builder) {
        this.make = builder.make;
        this.model = builder.model;
        this.year = builder.year;
        this.plateNumber = builder.plateNumber;
    }

    public static class VehicleBuilder {
        private String make;
        private String model;
        private int year;
        private String plateNumber;

        public VehicleBuilder(String make, String model) {
            this.make = make;
            this.model = model;
        }

        public VehicleBuilder setYear(int year) {
            this.year = year;
            return this;
        }

        public VehicleBuilder setPlateNumber(String plateNumber) {
            this.plateNumber = plateNumber;
            return this;
        }

        public Vehicle build() {
            return new Vehicle(this);
        }
    }
}
```

## 2. Large method

### 2.1. Problem

- Large methods make code harder to read and harder to unit test

```java
class Vehicle {
    private String make;
    private String model;
    private int year;
    private String color;
    private int mileage;

    // Constructors, getters, and setters

    public void printInfo() {
        System.out.println("Make: " + make);
        System.out.println("Model: " + model);
        System.out.println("Year: " + year);
        System.out.println("Color: " + color);
        System.out.println("Mileage: " + mileage);

        // Additional logic and operations specific to printing vehicle information ...
    }
}
```

### 2.2. Refactoring method

- Easier to read and unit tests (NOTE: Extracting print methods might not be a realistic refactoring example)

```java
class Vehicle {
    private String make;
    private String model;
    private int year;
    private String color;
    private int mileage;

    // Constructors, getters, and setters

    public void printInfo() {
        printMake();
        printModel();
        printYear();
        printColor();
        printMileage();

        // Additional logic and operations specific to printing vehicle information ...
    }

    private void printMake() {
        System.out.println("Make: " + make);
    }

    private void printModel() {
        System.out.println("Model: " + model);
    }

    private void printYear() {
        System.out.println("Year: " + year);
    }

    private void printColor() {
        System.out.println("Color: " + color);
    }

    private void printMileage() {
        System.out.println("Mileage: " + mileage);
    }
}
```

## 3. Complex conditional

### 3.1. Problem

- Complex conditionals are hard to read and understand

```java
class Vehicle {
    private int year;
    private double price;

    // Constructors, getters, and setters

    // Complex Condition - If the conditions are long, and there many conditions, it would be hard to read
    public void printTaxInformation() {
        if (year < 2010 && price > 20000) {
            double tax = price * 0.1;
            System.out.println("Tax to be paid: " + tax);
        } else {
            System.out.println("No tax to be paid.");
        }
    }
}
```

### 3.2. Refactoring method

- Simplify the condition with an explainatory variable. This won't force the readers to focus on the details.

```java
class Vehicle {
    private int year;
    private double price;

    // Constructors, getters, and setters

    public void printTaxInformation() {
        boolean isTaxApplicable = year < 2010 && price > 20000; // Explainatory Variable

        if (isTaxApplicable) {
            double tax = price * 0.1;
            System.out.println("Tax to be paid: " + tax);
        } else {
            System.out.println("No tax to be paid.");
        }
    }
}
```

## 4. Primitive Obsession

### 4.1. Problem

- An example of "primitive obsession," where the code relies on primitive types to represent a concept that could benefit from a dedicated custom object.

```java
class Vehicle {
    private String make;
    private String model;
    private int year;
    private String color;
    private int mileage;
    private String engineType;
    private int engineHorsepower;

    // Constructors, getters, and setters

    public void printInfo() {
        System.out.println("Make: " + make);
        System.out.println("Model: " + model);
        System.out.println("Year: " + year);
        System.out.println("Color: " + color);
        System.out.println("Mileage: " + mileage);
        System.out.println("Engine: " + engineType + " (" + engineHorsepower + " HP)");
    }
}
```

### 4.2. Refactoring method

```java
class Vehicle {
    private String make;
    private String model;
    private int year;
    private String color;
    private int mileage;
    private Engine engine;

    // Constructors, getters, and setters

    public void printInfo() {
        System.out.println("Make: " + make);
        System.out.println("Model: " + model);
        System.out.println("Year: " + year);
        System.out.println("Color: " + color);
        System.out.println("Mileage: " + mileage);
        System.out.println("Engine: " + engine.getDescription());
    }

    // Custom object representing the engine
    public static class Engine {
        private String type;
        private int horsepower;

        // Constructors, getters, and setters

        public String getDescription() {
            return type + " (" + horsepower + " HP)";
        }
    }
}
```

## 5. Large conditional

### 5.1. Problem

- Large conditionals are hard to read and understand

```java
class Vehicle {
    private String make;
    private String model;
    private int year;

    // Constructors, getters, and setters

    public void printInfo() {
        System.out.println("Make: " + make);
        System.out.println("Model: " + model);
        System.out.println("Year: " + year);

        if (year <= 2000) {
            System.out.println("Vintage Vehicle");
        } else if (year <= 2010) {
            System.out.println("Classic Vehicle");
        } else {
            System.out.println("Modern Vehicle");
        }
    }
}
```

### 5.2. Refactoring method

```java
abstract class Vehicle {
    private String make;
    private String model;
    private int year;

    public Vehicle(String make, String model, int year) {
        this.make = make;
        this.model = model;
        this.year = year;
    }

    public void printInfo() {
        System.out.println("Make: " + make);
        System.out.println("Model: " + model);
        System.out.println("Year: " + year);
        System.out.println("Type: " + getTypeLabel());
    }

    public abstract String getTypeLabel();
}

class VintageVehicle extends Vehicle {
    public VintageVehicle(String make, String model, int year) {
        super(make, model, year);
    }

    public String getTypeLabel() {
        return "Vintage Vehicle";
    }
}

class ClassicVehicle extends Vehicle {
    public ClassicVehicle(String make, String model, int year) {
        super(make, model, year);
    }

    public String getTypeLabel() {
        return "Classic Vehicle";
    }
}

class ModernVehicle extends Vehicle {
    public ModernVehicle(String make, String model, int year) {
        super(make, model, year);
    }

    public String getTypeLabel() {
        return "Modern Vehicle";
    }
}

public class Main {
    public static void main(String[] args) {
        ModernVehicle modernVehicle = new ModernVehicle("Ferrari", "Scuderia", 2022);
        modernVehicle.printInfo();
    }
}
```

## 6. Code duplication

### 6.1. Problem

- This duplication violates the DRY

```java
class Car {
    private String make;
    private String model;
    private int year;

    // Constructors, getters, and setters

    public void start() {
        // Code to start a vehicle
        System.out.println("Starting the vehicle...");
        // Code specific to starting a car
        System.out.println("Ignition sequence initiated for Car.");
        // Common code for all vehicles
        System.out.println("Vehicle engine started.");
    }

    public void stop() {
        // Code to stop a vehicle
        System.out.println("Stopping the vehicle...");
        // Code specific to stopping a car
        System.out.println("Applying brakes for Car.");
        // Common code for all vehicles
        System.out.println("Vehicle engine stopped.");
    }
}

class Truck {
    private String make;
    private String model;
    private int year;

    // Constructors, getters, and setters

    public void start() {
        // Code to start a vehicle
        System.out.println("Starting the vehicle...");
        // Code specific to starting a truck
        System.out.println("Ignition sequence initiated for Truck.");
        // Common code for all vehicles
        System.out.println("Vehicle engine started.");
    }

    public void stop() {
        // Code to stop a vehicle
        System.out.println("Stopping the vehicle...");
        // Code specific to stopping a Truck
        System.out.println("Applying brakes for Truck.");
        // Common code for all vehicles
        System.out.println("Vehicle engine stopped.");
    }
}
```

### 6.2. Refactoring method

- The refactored code eliminates code duplication by providing a common template for the `start` and `stop` methods in the `Vehicle` class. Each specific vehicle type only needs to implement its own behavior, resulting in cleaner and more maintainable code.
- The Template Method Design Pattern promotes code reuse, improves readability, and allows for easy extension and modification of the algorithm without modifying the core structure. It eliminates the need for duplicated code and ensures a consistent execution flow across different vehicle types.

```java
abstract class Vehicle {
    private String make;
    private String model;
    private int year;

    // Constructors, getters, and setters

    public final void start() {
        System.out.println("Starting the vehicle...");
        specificStart();
        commonStart();
    }

    public final void stop() {
        System.out.println("Stopping the vehicle...");
        specificStop();
        commonStop();
    }

    protected abstract void specificStart();

    protected abstract void specificStop();

    private void commonStart() {
        System.out.println("Vehicle engine started.");
    }

    private void commonStop() {
        System.out.println("Vehicle engine stopped.");
    }
}

class Car extends Vehicle {
    @Override
    protected void specificStart() {
        System.out.println("Ignition sequence initiated for Car.");
    }

    @Override
    protected void specificStop() {
        System.out.println("Applying brakes for Car.");
    }
}

class Truck extends Vehicle {
    @Override
    protected void specificStart() {
        System.out.println("Ignition sequence initiated for Truck.");
    }

    @Override
    protected void specificStop() {
        System.out.println("Applying brakes for Truck.");
    }
}
```
