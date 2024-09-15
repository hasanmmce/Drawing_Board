from PyQt5.QtWidgets import (
    QGraphicsView, QGraphicsScene, QGraphicsRectItem, QGraphicsEllipseItem, 
    QGraphicsLineItem, QGraphicsPolygonItem, QMainWindow, QVBoxLayout, QWidget, 
    QPushButton, QListWidget, QLineEdit, QHBoxLayout, QAction, QLabel, QApplication, 
    QGroupBox, QMenu, QTextEdit, QSplitter, QInputDialog, QFileDialog, 
     QGraphicsItem
)
from PyQt5.QtGui import QPainter, QPolygonF, QPen, QBrush
from PyQt5.QtCore import QRectF, QPointF, Qt, QTimer
import sys
import json


class DrawingCanvas(QGraphicsView):
    def __init__(self, main_window):
        super().__init__()
        self.main_window = main_window  # Store reference to the main window
        self.scene = QGraphicsScene()
        self.setScene(self.scene)
        self.setRenderHint(QPainter.Antialiasing)
        self.setSceneRect(QRectF(0, 0, 800, 600))  # Set the initial canvas size

        self.current_shape = None  # To hold the current shape being drawn
        self.start_point = None    # To record the initial click point
        self.current_item = None   # The item being drawn
        self.selected_item = None  # Track selected item for deletion
        self.shape_type = None     # The shape type to draw
        self.polygon_points = []   # List to store points for the polygon
        self.temp_polygon = None   # Temporary polygon item for dynamic drawing
        self.current_layer = None  # The current layer where objects are drawn
        self.grid_lines = []  # To store grid lines
        self.grid_spacing = 20  # Define grid spacing
        self.snap_marker = None  # To hold the temporary marker for snapping
        self.grid_visible = False  # Track whether grid is on or off
        self.first_point_selected = False  # Track if the first point has been clicked

    def resizeEvent(self, event):
        super().resizeEvent(event)
        # Update the scene rect based on the new size of the view
        self.setSceneRect(QRectF(0, 0, self.viewport().width(), self.viewport().height()))

        # Redraw the grid if grid lines are currently displayed
        if self.grid_visible:
            self.clear_grid()
            self.draw_grid()

    def toggle_grid(self, visible):
        self.grid_visible = visible
        if visible:
            self.draw_grid()
        else:
            self.clear_grid()

    def draw_grid(self):
        """Draw a grid with equally spaced lines."""
        spacing = self.grid_spacing  # Grid spacing
        width = int(self.sceneRect().width())
        height = int(self.sceneRect().height())

        # Draw vertical lines
        for x in range(0, width, spacing):
            line = self.scene.addLine(x, 0, x, height, QPen(Qt.lightGray))
            self.grid_lines.append(line)

        # Draw horizontal lines
        for y in range(0, height, spacing):
            line = self.scene.addLine(0, y, width, y, QPen(Qt.lightGray))
            self.grid_lines.append(line)

    def clear_grid(self):
        """Remove all grid lines."""
        for line in self.grid_lines:
            self.scene.removeItem(line)
        self.grid_lines = []

        # Also remove the snap marker when grid is off
        if self.snap_marker:
            self.scene.removeItem(self.snap_marker)
            self.snap_marker = None

    def mouseMoveEvent(self, event):
        super().mouseMoveEvent(event)

        # Get current mouse position
        mouse_pos = self.mapToScene(event.pos())

        # Calculate the nearest grid point
        nearest_grid_x = round(mouse_pos.x() / self.grid_spacing) * self.grid_spacing
        nearest_grid_y = round(mouse_pos.y() / self.grid_spacing) * self.grid_spacing
        nearest_point = QPointF(nearest_grid_x, nearest_grid_y)

        # Determine color for the snap marker (blue for the first point, red for subsequent points)
        snap_color = Qt.blue if not self.first_point_selected else Qt.red

        # Visualize the snapping point by drawing a small circle marker
        if self.snap_marker:
            self.scene.removeItem(self.snap_marker)  # Remove the old marker

        self.snap_marker = self.scene.addEllipse(
            nearest_point.x() - 3, nearest_point.y() - 3, 6, 6,
            QPen(snap_color), QBrush(snap_color)
        )

        # Handle the drawing logic if a shape is being drawn
        if self.current_item and self.shape_type:
            current_point = self.mapToScene(event.pos())
            if self.shape_type == 'rectangle' or self.shape_type == 'circle':
                self.current_item.setRect(QRectF(self.start_point, current_point).normalized())
            elif self.shape_type == 'line':
                self.current_item.setLine(self.start_point.x(), self.start_point.y(), current_point.x(), current_point.y())

    def set_shape_type(self, shape_type):
        self.shape_type = shape_type
        if shape_type != 'polygon':
            self.polygon_points.clear()  # Clear polygon points when switching away
        self.first_point_selected = False  # Reset the point tracking for a new shape

    def set_current_layer(self, layer):
        self.current_layer = layer

    def mousePressEvent(self, event):
        # Handle left-click for drawing shapes
        if event.button() == Qt.LeftButton and self.shape_type and self.current_layer:
            self.start_point = self.mapToScene(event.pos())

            # Snap the start point to the nearest grid point
            self.start_point.setX(round(self.start_point.x() / self.grid_spacing) * self.grid_spacing)
            self.start_point.setY(round(self.start_point.y() / self.grid_spacing) * self.grid_spacing)

            if self.shape_type == 'rectangle':
                self.current_item = QGraphicsRectItem(QRectF(self.start_point, self.start_point))
            elif self.shape_type == 'circle':
                self.current_item = QGraphicsEllipseItem(QRectF(self.start_point, self.start_point))
            elif self.shape_type == 'line':
                self.current_item = QGraphicsLineItem(self.start_point.x(), self.start_point.y(),
                                                      self.start_point.x(), self.start_point.y())
            elif self.shape_type == 'polygon':
                self.polygon_points.append(self.start_point)
                if len(self.polygon_points) > 1 and self.temp_polygon:
                    self.scene.removeItem(self.temp_polygon)
                self.temp_polygon = QGraphicsPolygonItem(QPolygonF(self.polygon_points))
                self.temp_polygon.setPen(QPen(Qt.black, 2))
                self.current_layer.addToGroup(self.temp_polygon)
                return

            # Set pen and add the shape to the current layer
            if self.current_item:
                self.current_item.setPen(QPen(Qt.black, 2))
                self.current_layer.addToGroup(self.current_item)
                self.main_window.update_status(f"{self.shape_type.capitalize()} drawn")

            # Mark that the first point has been selected
            self.first_point_selected = True

            # Clear the snap marker when a point is selected
            if self.snap_marker:
                self.scene.removeItem(self.snap_marker)
                self.snap_marker = None

        # Handle right-click for selecting and showing a context menu
        elif event.button() == Qt.RightButton:
            clicked_item = self.itemAt(event.pos())  # Get the item at the clicked position
            if clicked_item and isinstance(clicked_item, QGraphicsItem):
                self.selected_item = clicked_item
                self.show_context_menu(event.globalPos())  # Show the context menu

    def delete_selected_item(self):
        if self.selected_item:
            self.scene.removeItem(self.selected_item)
            self.selected_item = None  # Clear the selection after deletion

    # Show properties of the selected item
    def show_properties(self):
        if self.selected_item:
            # Logic to show a properties dialog
            print("Showing properties for the selected item")

    # Mouse release event to finalize the shape drawing
    def mouseReleaseEvent(self, event):
        if event.button() == Qt.LeftButton:
            self.current_item = None  # Reset after finishing the drawing

    # Mouse double-click event to finish polygon drawing
    def mouseDoubleClickEvent(self, event):
        if self.shape_type == 'polygon' and self.polygon_points:
            final_polygon = QGraphicsPolygonItem(QPolygonF(self.polygon_points))
            final_polygon.setPen(QPen(Qt.black, 2))
            self.current_layer.addToGroup(final_polygon)
            self.polygon_points.clear()
            if self.temp_polygon:
                self.scene.removeItem(self.temp_polygon)
                self.temp_polygon = None

    # Context menu to delete objects
    def show_context_menu(self, pos):
        context_menu = QMenu(self)

        # Add options to the context menu
        delete_action = QAction("Delete", self)
        delete_action.triggered.connect(self.delete_selected_item)
        context_menu.addAction(delete_action)

        properties_action = QAction("Properties", self)
        properties_action.triggered.connect(self.show_properties)
        context_menu.addAction(properties_action)

        # Show the context menu at the cursor's position
        context_menu.exec_(pos)










class MainWindow(QMainWindow):
    def __init__(self):
        super().__init__()
        self.setWindowTitle("Drawing with Layer Management")
        self.setGeometry(100, 100, 1200, 700)

        self.canvas = DrawingCanvas(self)
        self.layers = {}
        self.current_layer_name = None

        # Create a horizontal splitter to resize between the canvas and layer management + status section
        horizontal_splitter = QSplitter(Qt.Horizontal)

        # Create a vertical splitter to resize between layer management and the status section on the left
        vertical_splitter = QSplitter(Qt.Vertical)

        # Layer Management Panel
        self.layer_management_panel = self.create_layer_management()

        # Status section
        self.status_section = QTextEdit(self)
        self.status_section.setReadOnly(True)  # Make it read-only

        # Add layer management and status section to the vertical splitter
        vertical_splitter.addWidget(self.layer_management_panel)
        vertical_splitter.addWidget(self.status_section)

        # Set initial sizes: 3/4 for layer management, 1/4 for status section
        vertical_splitter.setSizes([3 * self.height() // 4, self.height() // 4])

        # Canvas Area with Buttons on Top
        canvas_widget = QWidget()
        canvas_layout = QVBoxLayout()

        # Add buttons above the canvas
        button_layout = QHBoxLayout()
        increase_button = QPushButton("+ Grid", self)
        increase_button.clicked.connect(self.increase_grid_size)
        button_layout.addWidget(increase_button)

        decrease_button = QPushButton("- Grid", self)
        decrease_button.clicked.connect(self.decrease_grid_size)
        button_layout.addWidget(decrease_button)

        # Add the button layout and the canvas to the same layout
        canvas_layout.addLayout(button_layout)
        canvas_layout.addWidget(self.canvas)
        canvas_widget.setLayout(canvas_layout)

        # Add vertical splitter (left) and canvas (right) to the horizontal splitter
        horizontal_splitter.addWidget(vertical_splitter)
        horizontal_splitter.addWidget(canvas_widget)

        # Set initial sizes: 1/3 for the left side, 2/3 for the canvas
        horizontal_splitter.setSizes([self.width() // 3, 2 * self.width() // 3])

        # Set the layout for the main window
        container_widget = QWidget()
        layout = QVBoxLayout(container_widget)
        layout.addWidget(horizontal_splitter)
        self.setCentralWidget(container_widget)

        self.create_toolbar()
        self.create_menubar()  # Call the menu bar creation function

        # Add event to trigger context menu on right-click
        self.layer_list.setContextMenuPolicy(Qt.CustomContextMenu)
        self.layer_list.customContextMenuRequested.connect(self.show_layer_context_menu)

    def increase_grid_size(self):
        """Increase the grid size by a certain step and redraw the grid."""
        self.canvas.grid_spacing += 10  # Increase by 10 (or any desired step)
        if self.canvas.grid_visible:
            self.canvas.clear_grid()
            self.canvas.draw_grid()
        self.update_status(f"Grid size increased to {self.canvas.grid_spacing}")

    def decrease_grid_size(self):
        """Decrease the grid size by a certain step and redraw the grid."""
        if self.canvas.grid_spacing > 10:  # Prevent the grid from becoming too small
            self.canvas.grid_spacing -= 10  # Decrease by 10 (or any desired step)
            if self.canvas.grid_visible:
                self.canvas.clear_grid()
                self.canvas.draw_grid()
            self.update_status(f"Grid size decreased to {self.canvas.grid_spacing}")
        else:
            self.update_status(f"Grid size is at minimum {self.canvas.grid_spacing}")



    
    # Method to update the status section
    def update_status(self, message):
        self.status_section.append(message)

    def create_layer_management(self):
        # Layer Management Section
        layer_management_group = QGroupBox("Layer Management")
        layer_management_layout = QVBoxLayout()

        # Add Layer Input
        self.layer_input = QLineEdit(self)
        self.layer_input.setPlaceholderText("Enter Layer Name")
        layer_management_layout.addWidget(self.layer_input)

        # Add Layer Button
        add_layer_button = QPushButton("Add Layer", self)
        add_layer_button.clicked.connect(self.add_layer)
        layer_management_layout.addWidget(add_layer_button)

        # Delete Layer Button
        delete_layer_button = QPushButton("Delete Layer", self)
        delete_layer_button.clicked.connect(self.delete_layer)
        layer_management_layout.addWidget(delete_layer_button)

        # Layer List
        self.layer_list = QListWidget(self)
        self.layer_list.clicked.connect(self.select_layer)
        layer_management_layout.addWidget(self.layer_list)

        # Layer Visibility Toggle
        toggle_visibility_button = QPushButton("Toggle Visibility", self)
        toggle_visibility_button.clicked.connect(self.toggle_layer_visibility)
        layer_management_layout.addWidget(toggle_visibility_button)

        layer_management_group.setLayout(layer_management_layout)
        return layer_management_group

    # Context menu for right-click on layers
    def show_layer_context_menu(self, pos):
        selected_item = self.layer_list.itemAt(pos)
        if selected_item:
            # Select the layer on right-click
            self.layer_list.setCurrentItem(selected_item)
            self.current_layer_name = selected_item.text()

            # Create the context menu
            context_menu = QMenu(self)

            rename_action = QAction("Rename", self)
            rename_action.triggered.connect(self.rename_layer)
            context_menu.addAction(rename_action)

            delete_action = QAction("Delete", self)
            delete_action.triggered.connect(self.delete_layer)
            context_menu.addAction(delete_action)

            properties_action = QAction("Properties", self)
            properties_action.triggered.connect(self.show_properties)
            context_menu.addAction(properties_action)

            # Show the context menu at the cursor's position
            context_menu.exec_(self.layer_list.mapToGlobal(pos))

    # Rename selected layer
    def rename_layer(self):
        selected_item = self.layer_list.currentItem()
        if selected_item:
            layer_name = selected_item.text()
            new_name, ok = QInputDialog.getText(self, "Rename Layer", f"Rename layer '{layer_name}' to:")
            if ok and new_name:
                selected_item.setText(new_name)
                self.layers[new_name] = self.layers.pop(layer_name)
                self.update_status(f"Layer '{layer_name}' renamed to '{new_name}'")

    def create_toolbar(self):
        toolbar = self.addToolBar("Shapes Toolbar")

        rect_action = QAction("Rectangle", self)
        rect_action.triggered.connect(lambda: self.canvas.set_shape_type('rectangle'))
        toolbar.addAction(rect_action)
        rect_action.triggered.connect(lambda: self.update_status("Rectangle tool selected"))

        line_action = QAction("Line", self)
        line_action.triggered.connect(lambda: self.canvas.set_shape_type('line'))
        toolbar.addAction(line_action)
        line_action.triggered.connect(lambda: self.update_status("Line tool selected"))

        circle_action = QAction("Circle", self)
        circle_action.triggered.connect(lambda: self.canvas.set_shape_type('circle'))
        toolbar.addAction(circle_action)
        circle_action.triggered.connect(lambda: self.update_status("Circle tool selected"))

        polygon_action = QAction("Polygon", self)
        polygon_action.triggered.connect(lambda: self.canvas.set_shape_type('polygon'))
        toolbar.addAction(polygon_action)
        polygon_action.triggered.connect(lambda: self.update_status("Polygon tool selected"))

    def add_layer(self):
        layer_name = self.layer_input.text()
        if layer_name and layer_name not in self.layers:
            new_layer = self.canvas.scene.createItemGroup([])  # Create an empty layer
            self.layers[layer_name] = new_layer
            self.layer_list.addItem(layer_name)
            self.layer_input.clear()
            self.update_status(f"Layer '{layer_name}' added")

    def delete_layer(self):
        selected_item = self.layer_list.currentItem()
        if selected_item:
            layer_name = selected_item.text()
            layer = self.layers.get(layer_name)
            if layer:
                self.canvas.scene.removeItem(layer)
                del self.layers[layer_name]
                self.layer_list.takeItem(self.layer_list.currentRow())
                self.update_status(f"Layer '{layer_name}' deleted")

    def select_layer(self):
        selected_item = self.layer_list.currentItem()
        if selected_item:
            self.current_layer_name = selected_item.text()
            self.canvas.set_current_layer(self.layers[self.current_layer_name])
            self.update_status(f"Layer '{self.current_layer_name}' selected")

    def toggle_layer_visibility(self):
        selected_item = self.layer_list.currentItem()
        if selected_item:
            layer_name = selected_item.text()
            layer = self.layers.get(layer_name)
            if layer:
                layer.setVisible(not layer.isVisible())
                visibility_status = "visible" if layer.isVisible() else "hidden"
                self.update_status(f"Layer '{layer_name}' is now {visibility_status}")

    def show_properties(self):
        selected_item = self.layer_list.currentItem()
        if selected_item:
            layer_name = selected_item.text()
            self.update_status(f"Showing properties for layer '{layer_name}'")

    def create_menubar(self):
        menubar = self.menuBar()

        # File Menu
        file_menu = menubar.addMenu('File')
        open_action = QAction('Open', self)
        open_action.triggered.connect(self.open_file)
        file_menu.addAction(open_action)

        save_action = QAction('Save', self)
        save_action.triggered.connect(self.save_file)
        file_menu.addAction(save_action)

        # View Menu
        view_menu = menubar.addMenu('View')
        toggle_grid_action = QAction('Show Grid', self, checkable=True)
        toggle_grid_action.triggered.connect(lambda: self.canvas.toggle_grid(toggle_grid_action.isChecked()))
        view_menu.addAction(toggle_grid_action)

    def open_file(self):
        """Open a custom file format with .iim extension and load the saved canvas state."""
        file_path, _ = QFileDialog.getOpenFileName(self, "Open File", "", "Infrastructure Information Model (*.iim)")
        if file_path:
            try:
                # Load the file data
                with open(file_path, "r") as file:
                    canvas_data = json.load(file)

                # Set grid spacing
                self.canvas.grid_spacing = canvas_data.get("grid_spacing", 20)
                self.canvas.clear_grid()
                if self.canvas.grid_visible:
                    self.canvas.draw_grid()

                # Clear existing layers and load new data
                self.layers.clear()
                self.layer_list.clear()

                # Load layers and shapes
                for layer_name, shapes in canvas_data.get("layers", {}).items():
                    new_layer = self.canvas.scene.createItemGroup([])
                    self.layers[layer_name] = new_layer
                    self.layer_list.addItem(layer_name)
                    
                    for shape in shapes:
                        if shape["type"] == "rectangle":
                            rect = QRectF(shape["x"], shape["y"], shape["width"], shape["height"])
                            rect_item = QGraphicsRectItem(rect)
                            new_layer.addToGroup(rect_item)
                        elif shape["type"] == "circle":
                            rect = QRectF(shape["x"], shape["y"], shape["width"], shape["height"])
                            circle_item = QGraphicsEllipseItem(rect)
                            new_layer.addToGroup(circle_item)
                        elif shape["type"] == "line":
                            line_item = QGraphicsLineItem(shape["x1"], shape["y1"], shape["x2"], shape["y2"])
                            new_layer.addToGroup(line_item)

                self.update_status(f"File opened: {file_path}")
            except Exception as e:
                self.update_status(f"Failed to open file: {str(e)}")

  

    def save_file(self):
        """Save the current state of the canvas to a custom file format with .iim extension."""
        file_path, _ = QFileDialog.getSaveFileName(self, "Save File", "", "Infrastructure Information Model (*.iim)")
        if file_path:
            if not file_path.endswith(".iim"):  # Ensure the file has the correct extension
                file_path += ".iim"
            try:
                # Collect the canvas data (grid size, layers, and shapes)
                canvas_data = {
                    "grid_spacing": self.canvas.grid_spacing,
                    "layers": {}
                }
                # Iterate through each layer and save the shapes in the layer
                for layer_name, layer in self.layers.items():
                    layer_data = []
                    for item in layer.childItems():
                        if isinstance(item, QGraphicsRectItem):
                            shape_type = "rectangle"
                            rect = item.rect()
                            shape_data = {"type": shape_type, "x": rect.x(), "y": rect.y(), "width": rect.width(), "height": rect.height()}
                        elif isinstance(item, QGraphicsEllipseItem):
                            shape_type = "circle"
                            rect = item.rect()
                            shape_data = {"type": shape_type, "x": rect.x(), "y": rect.y(), "width": rect.width(), "height": rect.height()}
                        elif isinstance(item, QGraphicsLineItem):
                            shape_type = "line"
                            line = item.line()
                            shape_data = {"type": shape_type, "x1": line.x1(), "y1": line.y1(), "x2": line.x2(), "y2": line.y2()}
                        else:
                            continue
                        layer_data.append(shape_data)
                    canvas_data["layers"][layer_name] = layer_data

                # Save the collected data to the selected file in JSON format
                with open(file_path, "w") as file:
                    json.dump(canvas_data, file)
                self.update_status(f"File saved: {file_path}")
            except Exception as e:
                self.update_status(f"Failed to save file: {str(e)}")





if __name__ == '__main__':
    app = QApplication(sys.argv)
    window = MainWindow()
    window.show()
    sys.exit(app.exec_())
