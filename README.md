# test2

```python
import sys
import numpy as np
import pandas as pd
import pyqtgraph as pg
from pyqtgraph.console import ConsoleWidget
import matplotlib.pyplot as plt
from pathlib import Path

from qtconsole.rich_jupyter_widget import RichJupyterWidget
from qtconsole.manager import QtKernelManager

from PySide6.QtCore import QRect
from PySide6.QtGui import QColor
from PySide6.QtWidgets import QApplication, QMainWindow, QAbstractItemView, QComboBox

from plotter2 import Ui_MainWindow

pg.setConfigOption('background', 'w')
pg.setConfigOption('foreground', 'k')
pg.setConfigOptions(antialias=True)

def get_rgb_from_hex(code):
    code_hex = code.replace("#", "")
    rgb = tuple(int(code_hex[i:i+2], 16) for i in (0, 2, 4))
    return QColor.fromRgb(rgb[0], rgb[1], rgb[2])

class AppData():
    def __init__(self):
        self.files = []
        self.cases = []
        self.submodels = []
        self.df_dict = {}


class MainWindow(QMainWindow, Ui_MainWindow):
    def __init__(self):
        super(MainWindow, self).__init__()
        self.setupUi(self)

        colors = np.load("tab10.npy")
        self.gpens = []
        for i in colors:
            self.gpens.append(pg.mkPen(color=(i[0], i[1], i[2]), width=2))
        
        # set initial listbox selection modes and radio button states
        self.listbox_files.setSelectionMode(QAbstractItemView.SelectionMode.ExtendedSelection)
        self.listbox_case.setSelectionMode(QAbstractItemView.SelectionMode.SingleSelection)
        self.listbox_submodel.setSelectionMode(QAbstractItemView.SelectionMode.ExtendedSelection)

        self.radio_1cms.setChecked(True)
        self.radio_mc1s.setChecked(False)
        self.radio_mpl_png.setChecked(True)
        self.radio_mpl_svg.setChecked(False)

        # defined button and radio button callbacks
        self.button_browse.clicked.connect(self.browse)
        self.button_load_data.clicked.connect(self.load_data)
        self.button_show_editor.clicked.connect(self.show_editor)
        self.button_show_qtconsole.clicked.connect(self.show_qtconsole)
        self.button_show_datatree.clicked.connect(self.show_datatree)
        self.button_mpl_open.clicked.connect(self.gen_matplotlib)
        # self.button_mpl_copy.clicked.connect(self.)
        # self.button_mpl_save.clicked.connect(self.)
        self.radio_1cms.clicked.connect(self.update_1cms_plot_mode)
        self.radio_mc1s.clicked.connect(self.update_mc1s_plot_mode)
        # self.radio_mpl_png.clicked.connect(self.)
        # self.radio_mpl_svg.clicked.connect(self.)

        # define listbox selection changed callbacks
        self.listbox_case.itemSelectionChanged.connect(self.update_plot)
        self.listbox_submodel.itemSelectionChanged.connect(self.update_plot)

    def make_jupyter_widget_with_kernel(self):
        """Start a kernel, connect to it, and create a RichJupyterWidget to use it
        """
        USE_KERNEL = 'python3'
        kernel_manager = QtKernelManager(kernel_name=USE_KERNEL)
        kernel_manager.start_kernel()

        kernel_client = kernel_manager.client()
        kernel_client.start_channels()

        jupyter_widget = RichJupyterWidget()
        jupyter_widget.kernel_manager = kernel_manager
        jupyter_widget.kernel_client = kernel_client
        return jupyter_widget
    
    def show_qtconsole(self):
        self.jupyter_widget = self.make_jupyter_widget_with_kernel()

    def show_datatree(self):
        x = self.pos().x()
        y = self.pos().y()
        self.datatree = pg.DataTreeWidget(data=app_data.__dict__)
        self.datatree.setGeometry(QRect(x, y, 600, 400))
        self.datatree.show()

    def show_editor(self):
        x = self.pos().x()
        y = self.pos().y()
        self.editor = ConsoleWidget(namespace=app_data.__dict__)
        self.editor.setGeometry(QRect(x, y, 600, 400))
        self.editor.show()

    def browse(self):
        dialog = pg.FileDialog.getExistingDirectory(self)
        path = Path(dialog)
        self.lineedit_folder.setText(dialog)
        files = path.glob("savefile_*.csv")
        for i, file in enumerate(files):
            self.listbox_files.insertItem(i, str(file))

    def load_data(self):
        sel_item = [x.text() for x in self.listbox_files.selectedItems()]
        
        app_data.files = sel_item

        for i in sel_item:
            app_data.df_dict[Path(i).stem] = pd.read_csv(i)
            app_data.submodels = app_data.submodels + list(app_data.df_dict[Path(i).stem].columns)
        app_data.cases = list(app_data.df_dict.keys())
        app_data.submodels = list(set(app_data.submodels))
        app_data.submodels.sort()
        self.listbox_case.addItems(app_data.df_dict.keys())
        self.listbox_submodel.addItems(app_data.submodels)

    def update_1cms_plot_mode(self):
        self.listbox_case.setSelectionMode(QAbstractItemView.SelectionMode.SingleSelection)
        self.listbox_submodel.setSelectionMode(QAbstractItemView.SelectionMode.ExtendedSelection)

        self.listbox_case.clearSelection()
        self.listbox_submodel.clearSelection()

    def update_mc1s_plot_mode(self):
        self.listbox_case.setSelectionMode(QAbstractItemView.SelectionMode.ExtendedSelection)
        self.listbox_submodel.setSelectionMode(QAbstractItemView.SelectionMode.SingleSelection)

        self.listbox_case.clearSelection()
        self.listbox_submodel.clearSelection()

    def update_plot(self):
        self.pg_widget.clear()
        self.table_envelope_selected.clear()

        sel_case = [x.text() for x in self.listbox_case.selectedItems()]
        sel_sub = [x.text() for x in self.listbox_submodel.selectedItems()]

        if self.radio_1cms.isChecked():
            for case in sel_case:
                for i, sub in enumerate(sel_sub):
                    self.pg_widget.plot(app_data.df_dict[case][sub], name=sub, pen=self.gpens[i])
            min = app_data.df_dict[sel_case[0]].loc[:, sel_sub].min(axis="columns").min()
            max = app_data.df_dict[sel_case[0]].loc[:, sel_sub].max(axis="columns").max()
            self.table_envelope_selected.setData({"min":[min], "max": [max]})
        elif self.radio_mc1s.isChecked():
            for sub in sel_sub:
                for i, case in enumerate(sel_case):
                    self.pg_widget.plot(app_data.df_dict[case][sub], name=case, pen=self.gpens[i])
            # min = np.min([app_data.df_dict[x].loc[:, sel_sub].min(axis="columns").min() for x in sel_case])
            # min = np.max([app_data.df_dict[x].loc[:, sel_sub].max(axis="columns").max() for x in sel_case])
            self.table_envelope_selected.setData({"min":[min], "max": [max]})
        legend = self.pg_widget.addLegend()
        


    def gen_matplotlib(self):
        sel_case = [x.text() for x in self.listbox_case.selectedItems()]
        sel_sub = [x.text() for x in self.listbox_submodel.selectedItems()]

        fig, ax = plt.subplots(figsize=(10, 6))

        if self.radio_1cms.isChecked():
            for case in sel_case:
                for sub in sel_sub:
                    ax.plot(app_data.df_dict[case][sub], label=sub)
                    ax.set_title(case)
                    ax.legend()
        elif self.radio_mc1s.isChecked():
            for sub in sel_sub:
                for case in sel_case:
                    ax.plot(app_data.df_dict[case][sub], label=case)
                    ax.set_title(sub)
                    ax.legend()

        ax.legend()
        fig.show()

app = QApplication(sys.argv)
app_data = AppData()
window = MainWindow()
window.show()

app.exec()

```
