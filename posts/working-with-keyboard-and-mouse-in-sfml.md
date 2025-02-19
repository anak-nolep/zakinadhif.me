---
title: 'Working with Keyboard and Mouse in SFML'
date: '2023-02-05'
lastEdited: '2023-02-08'
group: 'sfml-tidbits.txt'
tags: ['SFML', 'GameDev']
---

To write an interactive software, you need to be able to easily get what the user is clicking and typing in your game. In SFML, interactive events information are stored inside `sf::Event`, this class contains a type enum and a union of member variables. This allows SFML to store a variety of data with different structures inside of one class.

These events are sent by the OS by a mechanism that’s different between platforms, SFML abstracts this away from the user and provides them with a neat `sf::Event` and `sf::Window` abstraction.

Some example of events are:

  - Mouse released (`sf::Event::MouseReleased`)
  - Key pressed (`sf::Event::KeyReleased`)
  - Window resized (`sf::Event::WindowResized`)

Events are usually specifically sent by the OS to the corresponding window. That’s why you poll events from `sf::Window` instances.

And a good thing to keep in mind that there can be multiple events happening within a single frame. You should handle this correctly.

Here’s a snippet of code that handles some window events in SFML:

```cpp
int main() {
    sf::RenderWindow window({400, 400}, "My SFML Window");

    sf::Vector2f mousePosition {};

    while (window.isOpen()) {
        sf::Event event;
        while (window.pollEvent(event)) {
            switch (event.type) {
                case sf::Event::WindowClosed:
                    window.close();
                    break;
                case sf::Event::MouseMoved:
                    mousePosition = window.mapPixelToCoords({
                        event.mouseMove.x,
                        event.mouseMove.y
                    });
                    break;
                default: continue;
            }
        }
    }
}
```

## Why not use `isKeyPressed` and `isButtonPressed`?

In the past there has been some cases where SFML programs are falsely flagged as a virus by antiviruses because it uses the `isKeyPressed` function. The reason is, the same mechanism that `isKeyPressed` use is used by keyloggers to log keys because it listens to key presses even if the key presses are not intended for that particular window.

In retrospect, the SFML key pressed events are sent only if the window are focused. In MacOS, a special permission even needs to be granted by the user first for `isKeyPressed` to work.

`isButtonPressed` doesn't have the same restriction as `isKeyPressed` but it can cause subtle bugs, explanation are out of the scope of this article.

## Writing a user input interface

There are some things to consider when we want to write a user input interface. We need to be able to query:

  - `isUp`: Returns true if the key is up (released).
  - `isDown`: Returns true if the key is down (pressed).
  - `isPressed`: Checks if key has just been pressed in the current frame.
  - `isReleased`: Checks if key has just been released in the current frame.

The difference between the `isUp` and `isPressed` variants is that `isUp` checks for the current state of the button, yet `isPressed` measures the change in key state.

```c++
class Keyboard {
public:
    Keyboard() = delete;

    static bool isKeyPressed(sf::Keyboard::Key key);
    static bool isKeyReleased(sf::Keyboard::Key key);

    static bool isKeyUp(sf::Keyboard::Key key);
    static bool isKeyDown(sf::Keyboard::Key key);

private:
    static void setKey(sf::Keyboard::Key key, bool pressed);
    static void clearChangeInKeyStates();

    // Records change in key states
    static std::array<bool, sf::Keyboard::Key::KeyCount> m_justReleasedKeys;
    static std::array<bool, sf::Keyboard::Key::KeyCount> m_justPressedKeys;

    // Records key states
    static std::array<bool, sf::Keyboard::Key::KeyCount> m_keyStates;

    // Allows only the Engine class to modify Keyboard's member variables.
    friend class Engine;
};
```

Implementation:

```c++
bool Keyboard::isKeyReleased(sf::Keyboard::Key key) {
	return m_justReleasedKeys[key];
}

bool Keyboard::isKeyPressed(sf::Keyboard::Key key) {
	return m_justPressedKeys[key];
}

bool Keyboard::isKeyDown(sf::Keyboard::Key key) {
	return m_keyStates[key];
}

void Keyboard::setKey(sf::Keyboard::Key key, bool pressed) {
	m_keyStates[key] = pressed;

	if (pressed) m_justPressedKeys[key] = true;
	else m_justReleasedKeys[key] = true;
}

void Keyboard::clearKeyStates() {
	std::fill(m_justReleasedKeys.begin(), m_justReleasedKeys.end(), false);
	std::fill(m_justPressedKeys.begin(), m_justPressedKeys.end(), false);
}
```