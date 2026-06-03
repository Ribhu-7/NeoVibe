Role: Act as a Principal iOS Software Engineer specializing in scalable architecture and robust UI layouts.

Action: Generate the full, production-ready Swift code for fetching data from an API and rendering it inside a UICollectionView.

Context: I am building a feature that requires a strict separation of concerns. I want to use the MVVM (Model-View-ViewModel) design pattern. The network fetching, data arrays, and loading states must live inside the ViewModel, while the ViewController handles the layout. The layout constraints must be defined programmatically and perfectly anchored so the screen remains completely stable throughout the entire lifecycle.

Target: Give me two separate, fully implemented code blocks (the ViewModel and the ViewController) with all necessary Codable models, layout configurations, and data-binding code so I can copy-paste them directly into my project without missing pieces.

--------------------------------------------------------------

2.

Role: Act as an expert iOS Architect with deep mastery of reactive programming (Combine) and Clean Architecture principles.

Action: Write a complete, production-ready implementation in Swift showing how to fetch API data and propagate it to a View using Combine.

Context: I want a strict separation of concerns. The data must flow from a network service to a Network Repository/Use Case, then to a ViewModel, which exposes the state to a UIViewController. I want to use Combine's publishers, operators, and subscribers to manage this stream, handle errors gracefully, and ensure memory is managed properly with AnyCancellable stores

Target: Provide the full, clean Swift code organized into separate architectural layers:

1.The API Client/Repository (returning a Combine AnyPublisher).

2.The Use Case/Interactor layer.

3.The ViewModel (exposing @Published properties or custom Input/Output structs for the View to bind to). Make it fully complete and copy-pasteable.