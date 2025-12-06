---
title: Créer une bottom sheet personnalisée et fluide avec coordinateSpace(.global) en SwiftUI
pubDatetime: 2025-08-15T12:00:00Z
slug: custom-sheet-with-global-coordinate-space
featured: false
draft: false
tags:
  - SwiftUI
  - iOS
  - gesture
  - animation
description:
  Comment implémenter une bottom sheet custom avec une gesture fluide.
---

Pour ce premier article, je voudrais partager un problème récent, auquel j'ai été confronté, lié à une mauvaise compréhension de l’API `DragGesture(coordinateSpace:)` en SwiftUI.

## Le contexte

Dans mon application, j'avais besoin d'implémenter une bottom sheet avec les contraintes suivantes :

- Apparition sans animation dès l'affichage de la vue parente.
- Pas d'arrière-plan opaque.
- Deux positions fixes : repliée (par défaut) et étendue.
- Positions calculées dynamiquement.
- Le contenu de la bottom sheet s’adapte à la hauteur disponible.

## Première implémentation

J'ai créé une structure `DraggableBottomSheet` avec `DragGesture()` et des positions fixes.

<details class="details-block">
<summary>DraggableBottomSheet.swift</summary>

```swift
struct DraggableBottomSheet<Content: View>: View {
    let minHeight: CGFloat
    let maxHeight: CGFloat
    let content: Content
    
    @State private var height: CGFloat = 0
    @State private var dragStartHeight: CGFloat = 0
    private var backgroundColor: Color = .white
    
    init(
        minHeight: CGFloat,
        maxHeight: CGFloat,
        @ViewBuilder content: () -> Content
    ) {
        self.minHeight = minHeight
        self.maxHeight = maxHeight
        self.content = content()
    }

    var body: some View {
        VStack(alignment: .center, spacing: 0) {
            dragIndicator()
                .background(backgroundColor)
                .gesture(
                    DragGesture()
                        .onChanged { value in
                            // Store starting height on first drag frame
                            if dragStartHeight == 0 {
                                dragStartHeight = height
                            }
                            
                            // Apply drag with a 50px overshoot allowance
                            let proposedHeight = dragStartHeight - value.translation.height
                            let lowerBound = minHeight - 50
                            let upperBound = maxHeight + 50
                            height = min(
                                max(proposedHeight, lowerBound),
                                upperBound
                            )
                        }
                        .onEnded { _ in
                            // Snap to top or bottom
                            let midpoint = (maxHeight + minHeight) / 2
                            withAnimation(
                                .spring(
                                    response: 0.35,
                                    dampingFraction: 0.8
                                )
                            ) {
                                height = height > midpoint ? maxHeight : minHeight
                            }
                            dragStartHeight = 0
                        }
                )
            
            let computedHeight = height - dragIndicatorBlocHeight
            if computedHeight > 0 {
                content
                    .frame(height: computedHeight)
            }
        }
        .frame(maxWidth: .infinity)
        .frame(height: height)
        .background(backgroundColor)
        .clipShape(
            UnevenRoundedRectangle(cornerRadii: .init(topLeading: 40, topTrailing: 40))
        )
        .shadow(color: .black.opacity(0.05), radius: 10, x: 0, y: -2)
        .onChange(of: minHeight) { newValue in
            if height == 0 {
                height = newValue
            } else {
                withAnimation {
                    height = newValue
                }
            }
        }
    }
    
    // MARK: - Drag Indicator
    
    private let dragIndicatorSize: CGSize = CGSize(width: 64, height: 6)
    private let dragIndicatorTopPadding: CGFloat = 8
    private let dragIndicatorBottomPadding: CGFloat = 10
    private var dragIndicatorBlocHeight: CGFloat {
        dragIndicatorSize.height + dragIndicatorTopPadding + dragIndicatorBottomPadding
    }
    
    private func dragIndicator() -> some View {
        Color.gray.opacity(0.5)
            .frame(
                width: dragIndicatorSize.width,
                height: dragIndicatorSize.height
            )
            .clipShape(Capsule())
            .padding(.top, dragIndicatorTopPadding)
            .padding(.bottom, dragIndicatorBottomPadding)
            .frame(maxWidth: .infinity, maxHeight: dragIndicatorBlocHeight)
    }
}
```
</details>

<details class="details-block">
<summary>ContentView.swift</summary>

```swift
struct ContentView: View {
    var body: some View {
            ZStack(alignment: .top) {
                topContent()
                
                VStack {
                    Spacer()
                    DraggableBottomSheet(
                        minHeight: 350,
                        maxHeight: 700
                    ) {
                        sheetContent()
                    }
                }
            }
            .background(Color.gray.opacity(0.1))
            .ignoresSafeArea(.all, edges: .bottom)
    }
    
    private func topContent() -> some View {
        VStack(spacing: 0) {
            Color.white
                .frame(height: 100)
            Color.blue
                .frame(height: 100)
            Color.red
                .frame(height: 100)
        }
    }
    
    private func sheetContent() -> some View {
        ScrollView {
            LazyVStack(spacing: 0) {
                ForEach(0..<100) { index in
                    Text("Item \(index + 1)")
                        .padding()
                        .background(Color.clear)
                }
            }
        }
        .scrollIndicators(.hidden)
    }
}
```
</details>

## Le problème : tremblements lors du drag

La bottom sheet tremblait dès que je la tirais.

<video autoplay loop muted playsinline controls class="video-center">
    <source src="/assets/bottom-sheet-local.webm" type="video/webm">
</video>

## La solution : `coordinateSpace(.global)`

Après quelques recherches, j’ai découvert que l’utilisation de `DragGesture(coordinateSpace: .global)` suffisait à résoudre le problème.

Dans `DraggableBottomSheet.swift`, remplacer :

```swift
DragGesture()
```

Par :

```swift
DragGesture(coordinateSpace: .global)
```

Résultat :

<video autoplay loop muted playsinline controls class="video-center">
    <source src="/assets/bottom-sheet-global-fix.webm" type="video/webm">
</video>

## Comprendre la différence

### Documentation Apple

```swift
public enum CoordinateSpace {

    /// The global coordinate space at the root of the view hierarchy.
    case global

    /// The local coordinate space of the current view.
    case local

    /// A named reference to a view's local coordinate space.
    case named(AnyHashable)
}
```

À première vue, c’est un peu abstrait.

### Analyse

En ajoutant un print dans `.onChanged` :

```swift
.onChanged { value in
    print("Dragging: \(value.location.y)")
    [...]
}
```

Avec `.local` (défaut) :

```swift
Dragging: 0.0
Dragging: 9.0
Dragging: -0.6666717529296875
Dragging: 7.999994913736998
Dragging: -1.333343505859375
Dragging: 7.666661580403684
Dragging: -2.0000101725260038
Dragging: 7.3333282470703125
Dragging: -2.3333333333333144
Dragging: 6.999989827473996
```

Avec `.global` :

```swift
Dragging: 506.6666564941406
Dragging: 505.6666564941406
Dragging: 505.0
Dragging: 504.3333282470703
Dragging: 503.3333282470703
Dragging: 502.0
Dragging: 501.3333282470703
Dragging: 499.3333282470703
Dragging: 497.0
Dragging: 495.3333282470703
```

La différence clé est la suivante :

- `.local` : Les coordonnées sont mesurées par rapport à la vue en mouvement. La référence change à chaque frame, créant des "sauts" dans les valeurs.
- `.global` : Les coordonnées sont mesurées par rapport à l’écran. La référence reste fixe, les valeurs sont stables.

On peut donc vérifier facilement : avec `.global`, `value.location.y` est relatif à l’écran, tandis qu’avec `.local`, il est relatif à la vue elle-même.

## Bonus : positions dynamiques

Calculer dynamiquement les positions min/max :

1. Ajouter les propriétés `topContentHeight` et `whiteContentHeight`.

```swift
@State private var topContentHeight: CGFloat = 0
@State private var whiteContentHeight: CGFloat = 0
```

2. Utiliser `GeometryReader`.

```swift
GeometryReader { geometry in
    ZStack(alignment: .top) {
        topContent()
        
        VStack {
            Spacer()
            DraggableBottomSheet(
                minHeight: geometry.size.height - topContentHeight,
                maxHeight: geometry.size.height - whiteContentHeight
            ) {
                sheetContent()
            }
        }
    }
    .background(Color.gray.opacity(0.1))
}
.ignoresSafeArea(.all, edges: .bottom)
```

3. Mesurer les hauteurs des blocs souhaités.

```swift
private func topContent() -> some View {
    VStack(spacing: 0) {
        Color.white
            .frame(height: 100)
            .coordinateSpace(name: "whiteContent")
            .readSize(in: "whiteContent") { height in
                whiteContentHeight = height
            }
        Color.blue
            .frame(height: 100)
        Color.red
            .frame(height: 100)
    }
    .coordinateSpace(name: "topContent")
    .readSize(in: "topContent") { height in
        topContentHeight = height
    }
}
```

Résultat final :

<video autoplay loop muted playsinline controls class="video-center">
    <source src="/assets/bottom-sheet-global.webm" type="video/webm">
</video>

Vous pouvez accéder au code source via [ce lien](https://github.com/ikaros-nb/DraggableBottomSheet).