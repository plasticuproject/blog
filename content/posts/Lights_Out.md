---
layout: post
title: "corCTF2024 lights-out"
categories: corCTF
date: 2024-07-30
image: /images/lights_out/cor.jpg
description: Solution to corCTF2024 misc challenge "lights-out" using linear algebra.
tags: [corCTF, Python]
katex: true
markup: "markdown"
---

![Image](/images/lights_out/cor.jpg#center)

******

## The Challenge

![Image](/images/lights_out/challenge.png#center)

When we connect to the server, we are presented with some information about the game, how to play, an example, and the game board we need to solve. The game is pretty simple: the board is an N x N grid where we need to turn off all the "lights" by flipping, or not flipping, each position. The trick is that each time we modify a position, the adjacent positions will flip as well. The source code for the challenge was provided, but we will ignore that as it was unintinded. I know this because I'm the author of this challenge and was pissed when strellic accidentally included it as a download with the challenge. Arg.

![Image](/images/lights_out/console.png#center)

If we send an incorrect solution, a new game board is generated.

![Image](/images/lights_out/wrong.png#center)

After about 10 seconds, if you did not submit a solution, the game would time out and generate a new board. At least, this is how the challenge was supposed to work. Strellic reworked the code so that it would work for a redpwn jailed instance instead of a team instancer as I had intended, so instead it just shit the bed after 30 seconds and made you reconnect. This is what you were supposed to see:

![Image](/images/lights_out/timeout.png#center)

So we need to find a solution to the board very quickly and send that to the server to solve the challenge.

******

## Research

I am not a math wizard, but searching online I found that an optimal solution for certain Lights Out boards can be found by creating a linear algebra system and performing simple row operations. [Solving LightsOut using Linear Algebra](https://raw.org/research/solving-lightsout-using-linear-algebra/) is a great breakdown of the problem. We can represent each position of our board in a vector/array using binary values, *0* for off, and *1* for on. We then use [mod 2 addition](https://www.csus.edu/indiv/p/pangj/166/f/d8/5_Modulo%202%20Arithmetic.pdf) on each position, representing a position flip, to create a matrix of all position flips.
```python
def create_vector_representations(n: int) -> list[list[int]]:
    """
    Create vector representations for each position on the n x n board.

    Args:
        n (int): The size of the board (n x n).

    Returns:
        list[list[int]]: A list of vectors representing the effect of
                           toggling each light.
    """
    vectors = []
    for i in range(n * n):
        vector = [0] * (n * n)
        vector[i] = 1
        if i % n != 0:
            vector[i - 1] = 1  # left
        if i % n != n - 1:
            vector[i + 1] = 1  # right
        if i >= n:
            vector[i - n] = 1  # up
        if i < n * (n - 1):
            vector[i + n] = 1  # down
        vectors.append(vector)
    return vectors
```

Then we create an [augmented matrix](https://en.wikipedia.org/wiki/Augmented_matrix) using that matrix and the board state.
```python
def create_augmented_matrix(vectors: list[list[int]],
                            board: list[int]) -> list[list[int]]:
    """
    Create an augmented matrix from the vectors and board state.

    Args:
        vectors (list[list[int]]): The vector representations.
        board (list[int]): The current state of the board.

    Returns:
        list[list[int]]: The augmented matrix.
    """
    matrix = [vec + [board[i]] for i, vec in enumerate(vectors)]
    return matrix
```

Next, we preform [Gauss-Jordan Elimination](https://raw.org/book/linear-algebra/gauss-jordan-elimination/) on the given matrix to produce its [Reduced Row Echelon Form](https://en.wikipedia.org/wiki/Row_echelon_form#Reduced_row_echelon_form).
```python
def gauss_jordan_elimination(matrix: list[list[int]]) -> list[list[int]]:
    """
    Perform Gauss-Jordan elimination on the given matrix to produce its
    Reduced Row Echelon Form (RREF).

    Args:
        matrix (list[list[int]]): The matrix to be reduced.

    Returns:
        list[list[int]]: The matrix in RREF.
    """
    rows, cols = len(matrix), len(matrix[0])
    r = 0
    for c in range(cols - 1):
        if r >= rows:
            break
        pivot = None
        for i in range(r, rows):
            if matrix[i][c] == 1:
                pivot = i
                break
        if pivot is None:
            continue
        if r != pivot:
            matrix[r], matrix[pivot] = matrix[pivot], matrix[r]
        for i in range(rows):
            if i != r and matrix[i][c] == 1:
                for j in range(cols):
                    matrix[i][j] ^= matrix[r][j]
        r += 1
    return matrix
```

For good measure, we will check to make sure our augmented matrix is solvable.
```python
def is_solvable(matrix: list[list[int]]) -> bool:
    """
    Check if the given augmented matrix represents a solvable system.

    Args:
        matrix (list[list[int]]): The augmented matrix.

    Returns:
        bool: True if the system is solvable, False otherwise.
    """
    rref = gauss_jordan_elimination(matrix)
    for row in rref:
        if row[-1] == 1 and all(val == 0 for val in row[:-1]):
            return False
    return True

```

Finally, we will put it all together, creating our linear system and performing our operations to return our solution as a vector/array.
```python
def get_solution(board: list[int], n: int) -> list[int] | None:
    """
    Get a solution for the Lights Out board if it exists.

    Args:
        board (list[int]): The current state of the board.
        n (int): The size of the board (n x n).

    Returns:
        list[int] | None: A list representing the solution, or None
                            if no solution exists.
    """
    vectors = create_vector_representations(n)
    matrix = create_augmented_matrix(vectors, board)
    if not is_solvable(matrix):
        return None
    rref_matrix = gauss_jordan_elimination(matrix)
    return [row[-1] for row in rref_matrix[:n * n]]
```

Here is a great [video](https://youtu.be/1izbpSk3ays?si=6j2VksbgejLRrCJK) that explains and visualizes the process.

******

## Scripting

To solve the challenge, we need to connect to the server via TCP, find the latest board characters, and convert them to an array of *0*s and *1*s. Then we can pass that array to our **get_solution** function, convert the resulting array into a string using the *.* and *#* characters, send that to the server, then retrieve and print our flag.

```python
import re
import socket


def connect_to_server(host: str, port: int) -> None:
    """
    Connects to a server, reads the Lights Out board,
    solves it, sends the solution back, then retrieves
    and prints the flag.

    Args:
        host (str): The server's hostname or IP address.
        port (int): The server's port number.

    Returns:
        None
    """
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
        s.connect((host, port))
        print(f"Connected to server at {host}:{port}")

        try:
            buffer = ""
            while True:
                data = s.recv(1024).decode(
                    "utf-8")  # Read data from the server
                if not data:
                    break
                buffer += data

                # Extract the board from the received data
                matches = re.findall(
                    r"Lights Out Board:\n\n([\s\S]*?)\n\nYour Solution: ",
                    buffer)
                if matches:
                    board_str = matches[-1]
                    n = len(board_str.strip().split("\n"))
                    board = [
                        1 if char == "#" else 0 for char in board_str
                        if char in "#."
                    ]

                    # Solve the Lights Out board
                    solution = get_solution(board, n)

                    # Convert solution back to # and .
                    if solution:
                        solution_str = "".join("#" if val == 1 else "."
                                               for val in solution)

                        # Send the solution back to the server
                        s.sendall((solution_str + "\n").encode("utf-8"))

                        # Get and print flag
                        flag = s.recv(1024).decode("utf-8")
                        print(flag)

                    # Exit the loop after retrieving the flag
                    break

        except KeyboardInterrupt:
            print("Connection closed by user.")
        finally:
            s.close()


if __name__ == "__main__":
    SERVER_HOST = "be.ax"
    SERVER_PORT = 32421
    connect_to_server(SERVER_HOST, SERVER_PORT)

```

******

## Full Solution

```python
"""lights_out_solution.py"""
import re
import socket


def create_vector_representations(n: int) -> list[list[int]]:
    """
    Create vector representations for each position on the n x n board.

    Args:
        n (int): The size of the board (n x n).

    Returns:
        list[list[int]]: A list of vectors representing the effect of
                           toggling each light.
    """
    vectors = []
    for i in range(n * n):
        vector = [0] * (n * n)
        vector[i] = 1
        if i % n != 0:
            vector[i - 1] = 1  # left
        if i % n != n - 1:
            vector[i + 1] = 1  # right
        if i >= n:
            vector[i - n] = 1  # up
        if i < n * (n - 1):
            vector[i + n] = 1  # down
        vectors.append(vector)
    return vectors


def create_augmented_matrix(vectors: list[list[int]],
                            board: list[int]) -> list[list[int]]:
    """
    Create an augmented matrix from the vectors and board state.

    Args:
        vectors (list[list[int]]): The vector representations.
        board (list[int]): The current state of the board.

    Returns:
        list[list[int]]: The augmented matrix.
    """
    matrix = [vec + [board[i]] for i, vec in enumerate(vectors)]
    return matrix


def gauss_jordan_elimination(matrix: list[list[int]]) -> list[list[int]]:
    """
    Perform Gauss-Jordan elimination on the given matrix to produce its
    Reduced Row Echelon Form (RREF).

    Args:
        matrix (list[list[int]]): The matrix to be reduced.

    Returns:
        list[list[int]]: The matrix in RREF.
    """
    rows, cols = len(matrix), len(matrix[0])
    r = 0
    for c in range(cols - 1):
        if r >= rows:
            break
        pivot = None
        for i in range(r, rows):
            if matrix[i][c] == 1:
                pivot = i
                break
        if pivot is None:
            continue
        if r != pivot:
            matrix[r], matrix[pivot] = matrix[pivot], matrix[r]
        for i in range(rows):
            if i != r and matrix[i][c] == 1:
                for j in range(cols):
                    matrix[i][j] ^= matrix[r][j]
        r += 1
    return matrix


def is_solvable(matrix: list[list[int]]) -> bool:
    """
    Check if the given augmented matrix represents a solvable system.

    Args:
        matrix (list[list[int]]): The augmented matrix.

    Returns:
        bool: True if the system is solvable, False otherwise.
    """
    rref = gauss_jordan_elimination(matrix)
    for row in rref:
        if row[-1] == 1 and all(val == 0 for val in row[:-1]):
            return False
    return True


def get_solution(board: list[int], n: int) -> list[int] | None:
    """
    Get a solution for the Lights Out board if it exists.

    Args:
        board (list[int]): The current state of the board.
        n (int): The size of the board (n x n).

    Returns:
        list[int] | None: A list representing the solution, or None
                            if no solution exists.
    """
    vectors = create_vector_representations(n)
    matrix = create_augmented_matrix(vectors, board)
    if not is_solvable(matrix):
        return None
    rref_matrix = gauss_jordan_elimination(matrix)
    return [row[-1] for row in rref_matrix[:n * n]]


def connect_to_server(host: str, port: int) -> None:
    """
    Connects to a server, reads the Lights Out board,
    solves it, sends the solution back, then retrieves
    and prints the flag.

    Args:
        host (str): The server's hostname or IP address.
        port (int): The server's port number.

    Returns:
        None
    """
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
        s.connect((host, port))
        print(f"Connected to server at {host}:{port}")

        try:
            buffer = ""
            while True:
                data = s.recv(1024).decode(
                    "utf-8")  # Read data from the server
                if not data:
                    break
                buffer += data

                # Extract the board from the received data
                matches = re.findall(
                    r"Lights Out Board:\n\n([\s\S]*?)\n\nYour Solution: ",
                    buffer)
                if matches:
                    board_str = matches[-1]
                    n = len(board_str.strip().split("\n"))
                    board = [
                        1 if char == "#" else 0 for char in board_str
                        if char in "#."
                    ]

                    # Solve the Lights Out board
                    solution = get_solution(board, n)

                    # Convert solution back to # and .
                    if solution:
                        solution_str = "".join("#" if val == 1 else "."
                                               for val in solution)

                        # Send the solution back to the server
                        s.sendall((solution_str + "\n").encode("utf-8"))

                        # Get and print flag
                        flag = s.recv(1024).decode("utf-8")
                        print(flag)

                    # Exit the loop after retrieving the flag
                    break

        except KeyboardInterrupt:
            print("Connection closed by user.")
        finally:
            s.close()


if __name__ == "__main__":
    SERVER_HOST = "be.ax"
    SERVER_PORT = 32421
    connect_to_server(SERVER_HOST, SERVER_PORT)
```

We run our program and are greeted with the flag.

![Image](/images/lights_out/flag.png#center)

******
