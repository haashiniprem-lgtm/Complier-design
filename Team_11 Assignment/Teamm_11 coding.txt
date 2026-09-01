import tkinter as tk
from tkinter import ttk

# ============================================================
# ICU BIOMEDICAL COMPILER SIMULATOR
# Tynker / Python IDE - Single File
#
# Pipeline:
# LEX -> YACC-STYLE PARSER -> SDT -> SEMANTIC CHECK
#      -> 3AC / QUADRUPLES -> CSE -> LOW POWER MCU CODE
# ============================================================


# ------------------------------------------------------------
# 1. MEDICAL SYMBOL TABLE
# ------------------------------------------------------------

SYMBOL_TABLE = {
    "HR":      {"type": "INT",   "min": 30,  "max": 220, "unit": "bpm"},
    "SpO2":    {"type": "INT",   "min": 70,  "max": 100, "unit": "%"},
    "TEMP":    {"type": "FLOAT", "min": 30.0, "max": 43.0, "unit": "C"},
    "SYS":     {"type": "INT",   "min": 50,  "max": 200, "unit": "mmHg"},
    "DIA":     {"type": "INT",   "min": 30,  "max": 130, "unit": "mmHg"},
    "GLUCOSE": {"type": "FLOAT", "min": 40.0, "max": 400.0, "unit": "mg/dL"}
}

OPERATORS = {
    ">=": "GE",
    "<=": "LE",
    "==": "EQ",
    "!=": "NE",
    "&&": "AND",
    "||": "OR",
    ">": "GT",
    "<": "LT",
    "+": "PLUS",
    "-": "MINUS",
    "*": "MUL",
    "/": "DIV"
}


# ------------------------------------------------------------
# 2. LEXICAL ANALYZER
# ------------------------------------------------------------

def lexer(expression):
    tokens = []
    errors = []
    i = 0

    while i < len(expression):
        ch = expression[i]

        if ch.isspace():
            i += 1
            continue

        # Identifier
        if ch.isalpha() or ch == "_":
            start = i

            while i < len(expression):
                if expression[i].isalnum() or expression[i] == "_":
                    i += 1
                else:
                    break

            word = expression[start:i]

            if word in SYMBOL_TABLE:
                tokens.append(("ID", word))
            else:
                tokens.append(("UNKNOWN_ID", word))

            continue

        # Number
        if ch.isdigit() or ch == ".":
            start = i
            dots = 0

            while i < len(expression):
                if expression[i].isdigit():
                    i += 1
                elif expression[i] == ".":
                    dots += 1
                    if dots > 1:
                        break
                    i += 1
                else:
                    break

            text = expression[start:i]

            try:
                value = float(text)
                if value.is_integer():
                    value = int(value)
                tokens.append(("NUMBER", value))
            except:
                errors.append("Invalid number: " + text)

            continue

        # Parentheses
        if ch == "(":
            tokens.append(("LPAREN", ch))
            i += 1
            continue

        if ch == ")":
            tokens.append(("RPAREN", ch))
            i += 1
            continue

        # Two-character operators
        two = expression[i:i + 2]

        if two in OPERATORS:
            tokens.append((OPERATORS[two], two))
            i += 2
            continue

        # One-character operators
        if ch in OPERATORS:
            tokens.append((OPERATORS[ch], ch))
            i += 1
            continue

        errors.append(
            "Illegal character '" + ch +
            "' at position " + str(i)
        )
        i += 1

    return tokens, errors


# ------------------------------------------------------------
# 3. SYNTAX TREE
# ------------------------------------------------------------

class Node:
    def __init__(self, value, left=None, right=None):
        self.value = value
        self.left = left
        self.right = right


def tree_text(node, prefix="", last=True):
    if node is None:
        return ""

    result = prefix + ("`-- " if last else "|-- ")
    result += str(node.value) + "\n"

    new_prefix = prefix + ("    " if last else "|   ")

    children = []
    if node.left is not None:
        children.append(node.left)
    if node.right is not None:
        children.append(node.right)

    for index, child in enumerate(children):
        result += tree_text(
            child,
            new_prefix,
            index == len(children) - 1
        )

    return result


# ------------------------------------------------------------
# 4. YACC-STYLE RECURSIVE DESCENT PARSER
# ------------------------------------------------------------

class Parser:
    def __init__(self, tokens):
        self.tokens = tokens
        self.pos = 0
        self.errors = []

    def current(self):
        if self.pos < len(self.tokens):
            return self.tokens[self.pos]
        return ("EOF", "")

    def eat(self, expected):
        token = self.current()

        if token[0] == expected:
            self.pos += 1
            return token

        self.errors.append(
            "Expected " + expected +
            " but found '" + str(token[1]) + "'"
        )
        return None

    def parse(self):
        if not self.tokens:
            self.errors.append("Empty expression")
            return None

        root = self.expression()

        if self.current()[0] != "EOF":
            self.errors.append(
                "Unexpected token '" +
                str(self.current()[1]) + "'"
            )

        return root

    # expression -> AND { OR AND }
    def expression(self):
        node = self.and_expression()

        while self.current()[0] == "OR":
            op = self.eat("OR")[1]
            right = self.and_expression()
            node = Node(op, node, right)

        return node

    # AND { AND comparison }
    def and_expression(self):
        node = self.comparison()

        while self.current()[0] == "AND":
            op = self.eat("AND")[1]
            right = self.comparison()
            node = Node(op, node, right)

        return node

    # comparison -> arithmetic [comparison arithmetic]
    def comparison(self):
        node = self.arithmetic()

        while self.current()[0] in (
            "GT", "LT", "GE", "LE", "EQ", "NE"
        ):
            op = self.current()[1]
            self.pos += 1

            # Error recovery: operator must be followed by value
            if self.current()[0] in (
                "RPAREN", "AND", "OR", "EOF"
            ):
                self.errors.append(
                    "Missing right operand after '" +
                    str(op) + "'"
                )
                right = Node("ERROR")
            else:
                right = self.arithmetic()

            node = Node(op, node, right)

        return node

    # arithmetic -> term { + or - term }
    def arithmetic(self):
        node = self.term()

        while self.current()[0] in ("PLUS", "MINUS"):
            op = self.current()[1]
            self.pos += 1
            right = self.term()
            node = Node(op, node, right)

        return node

    # term -> factor { * or / factor }
    def term(self):
        node = self.factor()

        while self.current()[0] in ("MUL", "DIV"):
            op = self.current()[1]
            self.pos += 1

            if self.current()[0] in (
                "RPAREN", "AND", "OR", "EOF"
            ):
                self.errors.append(
                    "Missing right operand after '" +
                    str(op) + "'"
                )
                right = Node("ERROR")
            else:
                right = self.factor()

            node = Node(op, node, right)

        return node

    # factor -> ID | NUMBER | ( expression )
    def factor(self):
        token = self.current()

        if token[0] == "ID":
            self.pos += 1
            return Node(token[1])

        if token[0] == "UNKNOWN_ID":
            self.pos += 1
            self.errors.append(
                "Unknown identifier '" +
                str(token[1]) + "'"
            )
            return Node("ERROR")

        if token[0] == "NUMBER":
            self.pos += 1
            return Node(str(token[1]))

        if token[0] == "LPAREN":
            self.pos += 1
            node = self.expression()

            if self.current()[0] == "RPAREN":
                self.pos += 1
            else:
                self.errors.append(
                    "Missing closing ')'"
                )

            return node

        # Recovery
        self.errors.append(
            "Unexpected token '" +
            str(token[1]) + "'"
        )

        if token[0] != "EOF":
            self.pos += 1

        return Node("ERROR")


# ------------------------------------------------------------
# 5. TREE / ERROR HELPERS
# ------------------------------------------------------------

def contains_error(node):
    if node is None:
        return False

    if node.value == "ERROR":
        return True

    return contains_error(node.left) or contains_error(node.right)


def collect_identifiers(node, result=None):
    if result is None:
        result = []

    if node is None:
        return result

    if node.left is None and node.right is None:
        if node.value in SYMBOL_TABLE:
            if node.value not in result:
                result.append(node.value)
        return result

    collect_identifiers(node.left, result)
    collect_identifiers(node.right, result)

    return result


# ------------------------------------------------------------
# 6. SEMANTIC ANALYSIS
# ------------------------------------------------------------

def semantic_check(node):
    errors = []

    if node is None:
        return errors

    if node.left is None and node.right is None:
        if node.value == "ERROR":
            return errors

        if node.value in SYMBOL_TABLE:
            return errors

        try:
            float(node.value)
            return errors
        except:
            errors.append(
                "Invalid semantic value: " +
                str(node.value)
            )
            return errors

    semantic_check(node.left)
    semantic_check(node.right)

    return errors


def patient_range_check(entries):
    errors = []

    for vital, entry in entries.items():
        try:
            value = float(entry.get())
            info = SYMBOL_TABLE[vital]

            if value < info["min"] or value > info["max"]:
                errors.append(
                    vital + " = " + str(value) +
                    " is outside safe range " +
                    str(info["min"]) + " - " +
                    str(info["max"]) + " " +
                    info["unit"]
                )
        except:
            errors.append(
                vital + " contains a non-numeric value"
            )

    return errors


# ------------------------------------------------------------
# 7. THREE ADDRESS CODE + QUADRUPLES
# ------------------------------------------------------------

class ThreeAddress:
    def __init__(self):
        self.count = 0
        self.code = []
        self.quads = []

    def new_temp(self):
        self.count += 1
        return "t" + str(self.count)

    def generate(self, node):
        if node is None:
            return ""

        if node.left is None and node.right is None:
            return node.value

        left = self.generate(node.left)
        right = self.generate(node.right)

        temp = self.new_temp()

        self.code.append(
            temp + " = " +
            left + " " +
            node.value + " " +
            right
        )

        self.quads.append(
            (node.value, left, right, temp)
        )

        return temp


# ------------------------------------------------------------
# 8. COMMON SUBEXPRESSION ELIMINATION
# ------------------------------------------------------------

def optimize(code):
    optimized = []
    known = {}
    replacement = {}
    removed = []

    for instruction in code:
        parts = instruction.split()

        if len(parts) != 5:
            optimized.append(instruction)
            continue

        temp = parts[0]
        left = parts[2]
        op = parts[3]
        right = parts[4]

        if left in replacement:
            left = replacement[left]

        if right in replacement:
            right = replacement[right]

        key = (left, op, right)

        if key in known:
            replacement[temp] = known[key]
            removed.append(
                temp + " = " + left + " " +
                op + " " + right +
                "  -> reused " + known[key]
            )
        else:
            optimized.append(
                temp + " = " +
                left + " " +
                op + " " +
                right
            )
            known[key] = temp

    return optimized, removed


# ------------------------------------------------------------
# 9. LOW POWER TARGET CODE
# ------------------------------------------------------------

def target_code(code):
    assembly = []

    for instruction in code:
        parts = instruction.split()

        if len(parts) != 5:
            continue

        temp = parts[0]
        left = parts[2]
        op = parts[3]
        right = parts[4]

        assembly.append("LOAD R1, " + left)

        commands = {
            ">": "CMP_GT",
            "<": "CMP_LT",
            ">=": "CMP_GE",
            "<=": "CMP_LE",
            "==": "CMP_EQ",
            "!=": "CMP_NE",
            "&&": "AND",
            "||": "OR",
            "+": "ADD",
            "-": "SUB",
            "*": "MUL",
            "/": "DIV"
        }

        if op in commands:
            assembly.append(
                commands[op] + " R1, " + right
            )

        assembly.append(
            "MOV " + temp + ", R1"
        )

    assembly.append("CHECK_ALERT")
    assembly.append("LOW_POWER_SLEEP")

    return assembly


# ------------------------------------------------------------
# 10. GUI
# ------------------------------------------------------------

class CompilerGUI:

    def __init__(self, root):
        self.root = root
        self.root.title("ICU Biomedical Compiler Simulator")
        self.root.geometry("1400x850")
        self.root.minsize(1100, 700)
        self.root.configure(bg="#0d1820")

        self.last_data = {}

        self.build_header()
        self.build_patient_panel()
        self.build_expression_panel()
        self.build_step_buttons()
        self.build_output_area()
        self.build_footer()

    # --------------------------------------------------------
    # HEADER
    # --------------------------------------------------------

    def build_header(self):
        header = tk.Frame(
            self.root,
            bg="#0d1820"
        )
        header.pack(
            fill="x",
            padx=22,
            pady=(15, 8)
        )

        tk.Label(
            header,
            text="ICU  BIOMEDICAL COMPILER",
            font=("Arial", 25, "bold"),
            fg="#00d9ff",
            bg="#0d1820"
        ).pack(side="left")

        tk.Label(
            header,
            text="LEX / YACC / SDT / 3AC / CSE / MCU",
            font=("Arial", 11, "bold"),
            fg="#91a8b5",
            bg="#0d1820"
        ).pack(side="left", padx=25)

        self.status = tk.Label(
            header,
            text="● READY",
            font=("Arial", 12, "bold"),
            fg="#37df7a",
            bg="#0d1820"
        )
        self.status.pack(side="right")

    # --------------------------------------------------------
    # PATIENT PANEL
    # --------------------------------------------------------

    def build_patient_panel(self):
        frame = tk.LabelFrame(
            self.root,
            text="  LIVE PATIENT TELEMETRY  ",
            font=("Arial", 12, "bold"),
            fg="white",
            bg="#162832",
            bd=1
        )
        frame.pack(
            fill="x",
            padx=22,
            pady=6
        )

        self.patient_entries = {}

        values = {
            "HR": "112",
            "SpO2": "88",
            "TEMP": "38.7",
            "SYS": "145",
            "DIA": "92",
            "GLUCOSE": "126"
        }

        for vital, value in values.items():
            card = tk.Frame(
                frame,
                bg="#203945",
                width=170,
                height=75
            )
            card.pack(
                side="left",
                padx=8,
                pady=10
            )
            card.pack_propagate(False)

            tk.Label(
                card,
                text=vital,
                font=("Arial", 10, "bold"),
                fg="#9bb0bb",
                bg="#203945"
            ).pack(pady=(7, 0))

            entry = tk.Entry(
                card,
                width=9,
                justify="center",
                font=("Arial", 15, "bold"),
                bg="#203945",
                fg="white",
                insertbackground="white",
                relief="flat"
            )
            entry.insert(0, value)
            entry.pack(pady=3)

            self.patient_entries[vital] = entry

    # --------------------------------------------------------
    # EXPRESSION
    # --------------------------------------------------------

    def build_expression_panel(self):
        frame = tk.Frame(
            self.root,
            bg="#0d1820"
        )
        frame.pack(
            fill="x",
            padx=22,
            pady=7
        )

        tk.Label(
            frame,
            text="HEALTH RISK EXPRESSION",
            font=("Arial", 12, "bold"),
            fg="white",
            bg="#0d1820"
        ).pack(anchor="w")

        self.expression = tk.Entry(
            frame,
            font=("Consolas", 17, "bold"),
            bg="#182d38",
            fg="#00e5ff",
            insertbackground="white",
            relief="flat"
        )
        self.expression.pack(
            fill="x",
            pady=7,
            ipady=10
        )

        # Repeated HR > 100 intentionally demonstrates CSE
        self.expression.insert(
            0,
            "(HR > 100) && (SpO2 < 92) && (HR > 100)"
        )

        quick = tk.Frame(
            frame,
            bg="#0d1820"
        )
        quick.pack(fill="x")

        self.make_button(
            quick,
            "NORMAL TEST",
            self.normal_test,
            "#007f9e"
        ).pack(side="left", padx=(0, 8))

        self.make_button(
            quick,
            "CSE TEST",
            self.cse_test,
            "#376f8c"
        ).pack(side="left", padx=8)

        self.make_button(
            quick,
            "CORRUPTED INPUT",
            self.corrupt_test,
            "#934255"
        ).pack(side="left", padx=8)

        self.make_button(
            quick,
            "RANGE TEST",
            self.range_test,
            "#8b6b32"
        ).pack(side="left", padx=8)

        self.make_button(
            quick,
            "RESET",
            self.reset,
            "#465963"
        ).pack(side="right")

    # --------------------------------------------------------
    # STEP BUTTONS
    # --------------------------------------------------------

    def build_step_buttons(self):
        outer = tk.LabelFrame(
            self.root,
            text="  COMPILER PHASES - CLICK ANY STEP  ",
            font=("Arial", 12, "bold"),
            fg="white",
            bg="#162832",
            bd=1
        )
        outer.pack(
            fill="x",
            padx=22,
            pady=6
        )

        buttons = tk.Frame(
            outer,
            bg="#162832"
        )
        buttons.pack(
            fill="x",
            padx=10,
            pady=9
        )

        self.step_buttons = {}

        data = [
            ("1  LEX", self.show_lex, "#008cae"),
            ("2  YACC", self.show_yacc, "#008cae"),
            ("3  SDT", self.show_sdt, "#008cae"),
            ("4  SEMANTIC", self.show_semantic, "#008cae"),
            ("5  3AC", self.show_3ac, "#008cae"),
            ("6  CSE", self.show_cse, "#008cae"),
            ("7  MCU", self.show_mcu, "#008cae"),
            ("RUN ALL", self.compile_all, "#17935c")
        ]

        for name, command, color in data:
            button = self.make_button(
                buttons,
                name,
                command,
                color
            )
            button.pack(
                side="left",
                padx=4,
                expand=True,
                fill="x"
            )
            self.step_buttons[name] = button

    def make_button(self, parent, text, command, color):
        return tk.Button(
            parent,
            text=text,
            command=command,
            font=("Arial", 10, "bold"),
            bg=color,
            fg="white",
            activebackground="#00a8c6",
            activeforeground="white",
            relief="flat",
            bd=0,
            padx=10,
            pady=10,
            cursor="hand2"
        )

    # --------------------------------------------------------
    # OUTPUT AREA
    # --------------------------------------------------------

    def build_output_area(self):
        frame = tk.Frame(
            self.root,
            bg="#0d1820"
        )
        frame.pack(
            fill="both",
            expand=True,
            padx=22,
            pady=7
        )

        # Left side
        left = tk.LabelFrame(
            frame,
            text="  ANALYSIS / PARSE TREE / 3AC  ",
            font=("Arial", 12, "bold"),
            fg="white",
            bg="#162832",
            bd=1
        )
        left.pack(
            side="left",
            fill="both",
            expand=True,
            padx=(0, 6)
        )

        # Right side
        right = tk.LabelFrame(
            frame,
            text="  OPTIMIZATION / TARGET / RESULTS  ",
            font=("Arial", 12, "bold"),
            fg="white",
            bg="#162832",
            bd=1
        )
        right.pack(
            side="right",
            fill="both",
            expand=True,
            padx=(6, 0)
        )

        self.output = tk.Text(
            left,
            bg="#091217",
            fg="#d4e5ec",
            font=("Consolas", 12),
            relief="flat",
            wrap="none",
            padx=14,
            pady=12
        )
        self.output.pack(
            fill="both",
            expand=True,
            padx=8,
            pady=8
        )

        self.result = tk.Text(
            right,
            bg="#091217",
            fg="#d4e5ec",
            font=("Consolas", 12),
            relief="flat",
            wrap="none",
            padx=14,
            pady=12
        )
        self.result.pack(
            fill="both",
            expand=True,
            padx=8,
            pady=8
        )

    # --------------------------------------------------------
    # FOOTER
    # --------------------------------------------------------

    def build_footer(self):
        footer = tk.Frame(
            self.root,
            bg="#0d1820"
        )
        footer.pack(
            fill="x",
            padx=22,
            pady=(0, 10)
        )

        self.progress = tk.Label(
            footer,
            text="Pipeline: READY",
            font=("Arial", 10, "bold"),
            fg="#8fa7b3",
            bg="#0d1820"
        )
        self.progress.pack(side="left")

        tk.Label(
            footer,
            text="Biomedical safety compiler demonstration",
            font=("Arial", 9),
            fg="#637983",
            bg="#0d1820"
        ).pack(side="right")

    # --------------------------------------------------------
    # BASIC GUI HELPERS
    # --------------------------------------------------------

    def clear_outputs(self):
        self.output.delete("1.0", "end")
        self.result.delete("1.0", "end")

    def write_left(self, text):
        self.output.delete("1.0", "end")
        self.output.insert("end", text)

    def write_right(self, text):
        self.result.delete("1.0", "end")
        self.result.insert("end", text)

    def set_status(self, text, good=True):
        self.status.config(
            text=text,
            fg="#37df7a" if good else "#ff6577"
        )

    def set_progress(self, text):
        self.progress.config(text="Pipeline: " + text)

    # --------------------------------------------------------
    # BUILD DATA
    # --------------------------------------------------------

    def prepare(self):
        expression = self.expression.get()

        tokens, lex_errors = lexer(expression)
        parser = Parser(tokens)
        tree = parser.parse() if tokens else None

        semantic_errors = []
        range_errors = []

        if tree is not None:
            semantic_errors = semantic_check(tree)

        range_errors = patient_range_check(
            self.patient_entries
        )

        generator = None
        code = []
        quads = []
        optimized = []
        removed = []
        assembly = []

        if tree is not None and not parser.errors:
            generator = ThreeAddress()
            generator.generate(tree)
            code = generator.code
            quads = generator.quads
            optimized, removed = optimize(code)
            assembly = target_code(optimized)

        before_cycles = len(code) * 3 + 3
        after_cycles = len(optimized) * 3 + 3

        if before_cycles:
            saving = (
                (before_cycles - after_cycles)
                / before_cycles
            ) * 100
        else:
            saving = 0

        self.last_data = {
            "expression": expression,
            "tokens": tokens,
            "lex_errors": lex_errors,
            "parser": parser,
            "tree": tree,
            "semantic_errors": semantic_errors,
            "range_errors": range_errors,
            "code": code,
            "quads": quads,
            "optimized": optimized,
            "removed": removed,
            "assembly": assembly,
            "before_cycles": before_cycles,
            "after_cycles": after_cycles,
            "saving": saving
        }

        return self.last_data

    # --------------------------------------------------------
    # 1. LEX
    # --------------------------------------------------------

    def show_lex(self):
        data = self.prepare()

        text = "================================================\n"
        text += "              LEXICAL ANALYSIS\n"
        text += "================================================\n\n"

        text += "INPUT:\n"
        text += data["expression"] + "\n\n"

        text += "TOKEN TYPE       TOKEN VALUE\n"
        text += "---------------------------------------------\n"

        for token_type, value in data["tokens"]:
            text += f"{token_type:<16} {value}\n"

        if data["lex_errors"]:
            text += "\nLEXICAL ERRORS\n"
            for error in data["lex_errors"]:
                text += "X " + error + "\n"
        else:
            text += "\nOK - All characters tokenized successfully.\n"

        self.write_left(text)
        self.write_right(
            "LEX PURPOSE\n"
            "-----------\n"
            "The scanner converts the medical expression\n"
            "into tokens such as identifiers, numbers,\n"
            "operators and parentheses.\n\n"
            "Biomedical identifiers:\n"
            "HR, SpO2, TEMP, SYS, DIA, GLUCOSE\n"
        )

        self.set_status("● LEX COMPLETE")
        self.set_progress("LEX COMPLETE")

    # --------------------------------------------------------
    # 2. YACC
    # --------------------------------------------------------

    def show_yacc(self):
        data = self.prepare()

        text = "================================================\n"
        text += "             YACC-STYLE PARSER\n"
        text += "================================================\n\n"

        text += "GRAMMAR IDEA\n"
        text += "------------\n"
        text += "expression -> AND { OR AND }\n"
        text += "comparison -> arithmetic [comparison arithmetic]\n"
        text += "arithmetic -> term { + or - term }\n"
        text += "term       -> factor { * or / factor }\n"
        text += "factor     -> ID | NUMBER | ( expression )\n\n"

        if data["parser"].errors:
            text += "SYNTAX STATUS: ERROR\n\n"
            for error in data["parser"].errors:
                text += "X " + error + "\n"

            text += "\nRECOVERY:\n"
            text += "The parser identifies the bad location and\n"
            text += "continues where possible instead of crashing.\n"
        else:
            text += "SYNTAX STATUS: ACCEPTED\n"
            text += "OK - Expression matches the grammar.\n"

        self.write_left(text)

        if data["tree"] is not None:
            self.write_right(
                "SYNTAX TREE\n"
                "-----------\n\n" +
                tree_text(data["tree"])
            )
        else:
            self.write_right("No syntax tree generated.")

        self.set_status(
            "● SYNTAX ERROR" if data["parser"].errors
            else "● YACC COMPLETE",
            not bool(data["parser"].errors)
        )
        self.set_progress("YACC COMPLETE")

    # --------------------------------------------------------
    # 3. SDT / SYMBOL TABLE
    # --------------------------------------------------------

    def show_sdt(self):
        data = self.prepare()

        text = "================================================\n"
        text += "          SDT / MEDICAL SYMBOL TABLE\n"
        text += "================================================\n\n"

        text += (
            f"{'NAME':<12}"
            f"{'TYPE':<10}"
            f"{'MIN':<10}"
            f"{'MAX':<10}"
            f"UNIT\n"
        )
        text += "-" * 62 + "\n"

        for vital, info in SYMBOL_TABLE.items():
            text += (
                f"{vital:<12}"
                f"{info['type']:<10}"
                f"{info['min']:<10}"
                f"{info['max']:<10}"
                f"{info['unit']}\n"
            )

        text += "\nIDENTIFIERS USED IN EXPRESSION\n"
        text += "--------------------------------\n"

        if data["tree"] is not None:
            ids = collect_identifiers(data["tree"])
            for name in ids:
                text += (
                    name + " -> " +
                    SYMBOL_TABLE[name]["type"] +
                    "\n"
                )

        self.write_left(text)

        self.write_right(
            "SYNTAX-DIRECTED TRANSLATION\n"
            "---------------------------\n\n"
            "The parsed identifiers are connected to\n"
            "medical symbol-table attributes:\n\n"
            "TYPE       : INT / FLOAT\n"
            "MIN / MAX  : physiological boundary\n"
            "UNIT       : medical measurement unit\n\n"
            "These attributes are used during semantic\n"
            "validation and intermediate-code generation."
        )

        self.set_status("● SDT COMPLETE")
        self.set_progress("SDT COMPLETE")

    # --------------------------------------------------------
    # 4. SEMANTIC
    # --------------------------------------------------------

    def show_semantic(self):
        data = self.prepare()

        text = "================================================\n"
        text += "              SEMANTIC ANALYSIS\n"
        text += "================================================\n\n"

        if data["parser"].errors:
            text += "Semantic analysis blocked because syntax errors exist.\n"
        else:
            if data["semantic_errors"]:
                text += "TYPE / IDENTIFIER ERRORS\n"
                for error in data["semantic_errors"]:
                    text += "X " + error + "\n"
            else:
                text += "OK - Identifier and type validation passed.\n"

            text += "\nPATIENT RANGE CHECK\n"
            text += "-------------------\n"

            if data["range_errors"]:
                for error in data["range_errors"]:
                    text += "! " + error + "\n"
            else:
                text += "OK - All entered vital values are in range.\n"

        self.write_left(text)

        right = "SAFETY CHECK SUMMARY\n"
        right += "--------------------\n\n"

        for vital, entry in self.patient_entries.items():
            try:
                value = float(entry.get())
                info = SYMBOL_TABLE[vital]

                if info["min"] <= value <= info["max"]:
                    state = "SAFE"
                else:
                    state = "OUT OF RANGE"

                right += (
                    f"{vital:<10} {value:<8} "
                    f"{state}\n"
                )
            except:
                right += f"{vital:<10} INVALID\n"

        right += "\nA range violation produces a warning\n"
        right += "instead of silently accepting unsafe telemetry."

        self.write_right(right)

        self.set_status("● SEMANTIC COMPLETE")
        self.set_progress("SEMANTIC CHECK COMPLETE")

    # --------------------------------------------------------
    # 5. 3AC
    # --------------------------------------------------------

    def show_3ac(self):
        data = self.prepare()

        text = "================================================\n"
        text += "        THREE ADDRESS CODE / QUADRUPLES\n"
        text += "================================================\n\n"

        if data["parser"].errors:
            text += "3AC NOT GENERATED.\n"
            text += "Reason: syntax errors must be fixed first.\n"
        else:
            text += "THREE ADDRESS CODE\n"
            text += "------------------\n"

            for number, line in enumerate(data["code"], 1):
                text += f"{number:02d}. {line}\n"

            text += "\nQUADRUPLES\n"
            text += "----------\n"
            text += (
                f"{'#':<4}{'OP':<8}"
                f"{'ARG1':<10}{'ARG2':<10}"
                f"RESULT\n"
            )
            text += "-" * 48 + "\n"

            for number, quad in enumerate(
                data["quads"], 1
            ):
                op, arg1, arg2, result = quad
                text += (
                    f"{number:<4}"
                    f"{op:<8}"
                    f"{arg1:<10}"
                    f"{arg2:<10}"
                    f"{result}\n"
                )

        self.write_left(text)

        self.write_right(
            "3AC PURPOSE\n"
            "-----------\n\n"
            "Three Address Code breaks a complex medical\n"
            "expression into simple instructions.\n\n"
            "Example:\n"
            "t1 = HR > 100\n"
            "t2 = SpO2 < 92\n"
            "t3 = t1 && t2\n\n"
            "Each instruction uses at most two operands\n"
            "and one result temporary."
        )

        self.set_status("● 3AC COMPLETE")
        self.set_progress("3AC GENERATED")

    # --------------------------------------------------------
    # 6. CSE OPTIMIZATION
    # --------------------------------------------------------

    def show_cse(self):
        data = self.prepare()

        text = "================================================\n"
        text += "       COMMON SUBEXPRESSION ELIMINATION\n"
        text += "================================================\n\n"

        if data["parser"].errors:
            text += "Optimization unavailable until syntax is valid.\n"
            self.write_left(text)
            self.write_right("Fix the syntax error first.")
            self.set_status("● CSE BLOCKED", False)
            return

        text += "BEFORE OPTIMIZATION\n"
        text += "-------------------\n"

        for line in data["code"]:
            text += line + "\n"

        text += "\nAFTER CSE\n"
        text += "---------\n"

        for line in data["optimized"]:
            text += line + "\n"

        text += "\nOPTIMIZATION LOG\n"
        text += "----------------\n"

        if data["removed"]:
            for item in data["removed"]:
                text += "OK " + item + "\n"
        else:
            text += "No redundant expression found.\n"

        self.write_left(text)

        right = "CYCLE COMPARISON\n"
        right += "----------------\n\n"
        right += (
            "Before Optimization : " +
            str(data["before_cycles"]) +
            " cycles\n"
        )
        right += (
            "After Optimization  : " +
            str(data["after_cycles"]) +
            " cycles\n"
        )
        right += (
            "Cycle Reduction     : " +
            f"{data['saving']:.1f}%\n"
        )
        right += (
            "Instructions Removed: " +
            str(len(data["removed"])) +
            "\n"
        )

        right += "\nWHY CSE?\n"
        right += "Repeated calculations are reused instead of\n"
        right += "being evaluated again, reducing execution work."

        self.write_right(right)

        self.set_status("● CSE COMPLETE")
        self.set_progress("CSE OPTIMIZATION COMPLETE")

    # --------------------------------------------------------
    # 7. MCU TARGET
    # --------------------------------------------------------

    def show_mcu(self):
        data = self.prepare()

        text = "================================================\n"
        text += "          LOW POWER MCU TARGET CODE\n"
        text += "================================================\n\n"

        if data["parser"].errors:
            text += "Target code not generated because parsing failed.\n"
        else:
            text += "MICROCONTROLLER INSTRUCTIONS\n"
            text += "-----------------------------\n\n"

            for number, line in enumerate(
                data["assembly"], 1
            ):
                text += f"{number:02d}. {line}\n"

        self.write_left(text)

        right = (
            "TARGET PROFILE\n"
            "--------------\n\n"
            "Architecture : Simplified MCU\n"
            "Mode         : LOW POWER\n"
            "Registers    : R1 + temporaries\n"
            "Optimization  : CSE\n\n"
            "FINAL EXECUTION\n"
            "----------------\n"
            "Before : " +
            str(data["before_cycles"]) +
            " cycles\n"
            "After  : " +
            str(data["after_cycles"]) +
            " cycles\n\n"
            "The generated instructions are a\n"
            "simulation of low-power target code."
        )

        self.write_right(right)

        self.set_status("● MCU CODE READY")
        self.set_progress("TARGET CODE GENERATED")

    # --------------------------------------------------------
    # FULL COMPILATION
    # --------------------------------------------------------

    def compile_all(self):
        data = self.prepare()
        self.clear_outputs()

        # If syntax is broken, show useful recovery information
        if data["parser"].errors:
            self.write_left(
                "================================================\n"
                "            COMPILATION REPORT\n"
                "================================================\n\n"
                "STATUS: SYNTAX ERROR\n\n"
                "RECOVERY DETAILS\n"
                "----------------\n"
                + "\n".join(
                    "X " + e
                    for e in data["parser"].errors
                )
                + "\n\n"
                "The compiler stopped before 3AC generation\n"
                "because invalid syntax must not reach the\n"
                "optimization or target-code phases.\n"
            )

            self.write_right(
                "CORRUPTED TELEMETRY DETECTED\n"
                "-----------------------------\n\n"
                "LEX  : processed\n"
                "YACC : error detected\n"
                "SDT  : not executed\n"
                "3AC  : blocked\n"
                "CSE  : blocked\n"
                "MCU  : blocked\n\n"
                "RECOVERY:\n"
                "The parser identified the missing operand\n"
                "and prevented invalid intermediate code."
            )

            self.set_status(
                "● SYNTAX ERROR - RECOVERED",
                False
            )
            self.set_progress(
                "ERROR RECOVERY COMPLETE"
            )
            return

        # Normal full report
        left = "================================================\n"
        left += "              FULL COMPILER REPORT\n"
        left += "================================================\n\n"

        left += "1. LEXICAL ANALYSIS\n"
        left += "--------------------\n"
        left += "OK - " + str(len(data["tokens"])) + " tokens recognized.\n\n"

        left += "2. YACC-STYLE PARSING\n"
        left += "---------------------\n"
        left += "OK - Grammar accepted.\n\n"

        left += "3. SYNTAX TREE\n"
        left += "---------------\n"
        left += tree_text(data["tree"]) + "\n"

        left += "4. SDT / SYMBOL TABLE\n"
        left += "---------------------\n"

        for vital, info in SYMBOL_TABLE.items():
            left += (
                f"{vital:<10}"
                f"{info['type']:<8}"
                f"{info['min']:<7}"
                f"{info['max']:<7}"
                f"{info['unit']}\n"
            )

        left += "\n5. SEMANTIC CHECK\n"
        left += "------------------\n"
        left += "OK - Type and identifier validation passed.\n"

        if data["range_errors"]:
            left += "\nRANGE WARNINGS:\n"
            for error in data["range_errors"]:
                left += "! " + error + "\n"
        else:
            left += "OK - Telemetry values are within ranges.\n"

        left += "\n6. THREE ADDRESS CODE\n"
        left += "---------------------\n"
        for line in data["code"]:
            left += line + "\n"

        self.write_left(left)

        right = "================================================\n"
        right += "          OPTIMIZATION + TARGET REPORT\n"
        right += "================================================\n\n"

        right += "7. BEFORE CSE\n"
        right += "-------------\n"
        for line in data["code"]:
            right += line + "\n"

        right += "\n8. AFTER CSE\n"
        right += "------------\n"
        for line in data["optimized"]:
            right += line + "\n"

        right += "\nCSE RESULT\n"
        right += "----------\n"
        right += (
            "Redundant expressions removed: " +
            str(len(data["removed"])) + "\n"
        )

        for item in data["removed"]:
            right += "OK " + item + "\n"

        right += "\n9. EXECUTION CYCLES\n"
        right += "-------------------\n"
        right += (
            "Before Optimization : " +
            str(data["before_cycles"]) +
            " cycles\n"
        )
        right += (
            "After Optimization  : " +
            str(data["after_cycles"]) +
            " cycles\n"
        )
        right += (
            "Reduction           : " +
            f"{data['saving']:.1f}%\n"
        )

        right += "\n10. LOW POWER MCU CODE\n"
        right += "----------------------\n"
        for line in data["assembly"]:
            right += line + "\n"

        right += "\n============================================\n"
        right += "        COMPILATION SUCCESSFUL\n"
        right += "============================================\n"

        self.write_right(right)

        if data["range_errors"]:
            self.set_status(
                "● COMPILED - RANGE WARNING"
            )
        else:
            self.set_status(
                "● COMPILATION SUCCESS"
            )

        self.set_progress(
            "ALL PHASES COMPLETE"
        )

    # --------------------------------------------------------
    # TEST CASES
    # --------------------------------------------------------

    def normal_test(self):
        self.expression.delete(0, "end")
        self.expression.insert(
            0,
            "(HR > 100) && (SpO2 < 92)"
        )
        self.set_status("● NORMAL TEST LOADED")

    def cse_test(self):
        self.expression.delete(0, "end")
        self.expression.insert(
            0,
            "(HR > 100) && (SpO2 < 92) && (HR > 100)"
        )
        self.set_status("● CSE TEST LOADED")

    def corrupt_test(self):
        self.expression.delete(0, "end")
        self.expression.insert(
            0,
            "(HR > 100) && (SpO2 < )"
        )
        self.set_status(
            "● CORRUPTED INPUT LOADED",
            False
        )

    def range_test(self):
        self.expression.delete(0, "end")
        self.expression.insert(
            0,
            "(HR > 100) && (SpO2 < 92)"
        )

        self.patient_entries["SpO2"].delete(0, "end")
        self.patient_entries["SpO2"].insert(0, "130")

        self.set_status(
            "● RANGE TEST LOADED",
            False
        )

    def reset(self):
        self.expression.delete(0, "end")
        self.expression.insert(
            0,
            "(HR > 100) && (SpO2 < 92) && (HR > 100)"
        )

        defaults = {
            "HR": "112",
            "SpO2": "88",
            "TEMP": "38.7",
            "SYS": "145",
            "DIA": "92",
            "GLUCOSE": "126"
        }

        for vital, value in defaults.items():
            self.patient_entries[vital].delete(0, "end")
            self.patient_entries[vital].insert(0, value)

        self.clear_outputs()
        self.set_status("● READY")
        self.set_progress("READY")


# ------------------------------------------------------------
# START PROGRAM
# ------------------------------------------------------------

root = tk.Tk()
app = CompilerGUI(root)
root.mainloop()
