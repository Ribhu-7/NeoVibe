1. [Role]
Act as an expert iOS developer and senior Swift architect.

[Task]
Generate the complete code for a basic iOS UI that displays a list of furniture products using a UITableView and custom UITableViewCell. 

[Context]
I am an iOS developer building a programmatic UI (no Storyboards/XIBs) in Swift using UIKit. 
- The data model should be a simple `Product` struct with `name` (String) and `price` (String).
- The mock data array should include items like "Chair", "Table", "Sofa", and "Desk".
- The custom cell should display the product name on the left and the price on the right.
- Ensure proper Auto Layout constraints are set up programmatically, and use modern cell registration if applicable, or standard dequeuing.

[Format]
Please provide the output in clean, well-commented Swift code split into three distinct sections:
1. The Product data model and mock data.
2. The Custom UITableViewCell class with Auto Layout constraints.
3. The UIViewController class implementing UITableViewDataSource and UITableViewDelegate.


--------------------------------------------------------------

2. [Role]
Act as an expert iOS developer and senior Swift architect specializing in network layers and data serialization.

[Task]
Write a robust Network Manager in Swift to fetch product data from a mock API URL, decode the JSON response, and map it into a local data model.

[Context]
I am an iOS developer building a modern Swift application. 
- Use the modern `async/await` concurrency model (`URLSession.shared.data(from:)`).
- The JSON response is an array of objects, where each object looks like this: `{"id": 1, "product_name": "Chair", "price_usd": 299.99}`.
- The local Swift data model (`Product`) should conform to `Decodable` and use `CodingKeys` to map the snake_case JSON keys to camelCase Swift properties (`productName`, `priceUsd`).
- Include basic error handling using a custom `NetworkError` enum (e.g., `.invalidURL`, `.decodingError`, `.serverError`).

[Format]
Please provide the output in clean, well-commented Swift code split into three distinct sections:
1. The custom `NetworkError` enum and the `Product` data model with `CodingKeys`.
2. A generic or specific `NetworkManager` class/actor handling the API request.
3. An example snippet showing how to call this network manager from a UIViewController or ViewModel using a `Task` block.