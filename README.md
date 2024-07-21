# BaseRecyclerView
viewbinding kullanılan kapsamlı bir base recyclerview yapısı

## Kaynak Kod

```
abstract class BaseRecyclerView<T> @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null,
    style: Int = 0,
) : RecyclerView(context, attrs, style) {

    init {
        layoutManager = LinearLayoutManager(context)
        adapter = CustomAdapter()
    }


    abstract val itemLayoutId: Int
    abstract val currentList: ArrayList<T>
    abstract fun bind(viewHolderBinding: ViewDataBinding, item: T)

    open fun onRowMoved(from: Int, to: Int) {}
    open fun onRowSelected(viewHolder: CustomViewHolder) {}
    open fun onRowClear(viewHolder: CustomViewHolder) {}

    fun initDragDrop(recyclerView: BaseRecyclerView<*>, enable: Boolean) {
        val helper = MoveHelper(adapter as BaseRecyclerView<*>.CustomAdapter, enable)
        ItemTouchHelper(helper).attachToRecyclerView(recyclerView)
    }

    inner class CustomAdapter : Adapter<CustomViewHolder>(), RowTouchHelper {
        override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): CustomViewHolder {
            val binding = DataBindingUtil.inflate<ViewDataBinding>(
                LayoutInflater.from(parent.context),
                itemLayoutId,
                parent,
                false
            )
            return CustomViewHolder(binding)
        }

        override fun onBindViewHolder(holder: CustomViewHolder, position: Int) {
            bind(holder.binding, currentList[position])
        }

        override fun getItemCount() = currentList.size

        override fun rowMoved(from: Int, to: Int) {
            Collections.swap(currentList, from, to)
            notifyItemMoved(from, to)
            onRowMoved(from, to)
        }

        override fun rowSelected(viewHolder: CustomViewHolder) {
            onRowSelected(viewHolder)
        }

        override fun rowClear(viewHolder: CustomViewHolder) {
            onRowClear(viewHolder)
        }

    }

    class CustomViewHolder(val binding: ViewDataBinding) : ViewHolder(binding.root)


    // Drag drop reorder

    interface RowTouchHelper {
        fun rowMoved(from: Int, to: Int)
        fun rowSelected(viewHolder: CustomViewHolder)
        fun rowClear(viewHolder: CustomViewHolder)
    }


    inner class MoveHelper(private val helper: RowTouchHelper, private val enable: Boolean) :
        ItemTouchHelper.Callback() {

        override fun isLongPressDragEnabled() = false

        override fun isItemViewSwipeEnabled() = true

        override fun onSelectedChanged(viewHolder: ViewHolder?, actionState: Int) {
            super.onSelectedChanged(viewHolder, actionState)

            if (actionState != ItemTouchHelper.ACTION_STATE_IDLE)
                (viewHolder as? CustomViewHolder)?.let { helper.rowSelected(it) }
        }

        override fun getMovementFlags(recyclerView: RecyclerView, viewHolder: ViewHolder): Int {
            val dragFlags = if (enable) ItemTouchHelper.UP and ItemTouchHelper.DOWN else 0
            return makeMovementFlags(dragFlags, ItemTouchHelper.LEFT)
        }

        override fun onMove(
            recyclerView: RecyclerView,
            viewHolder: ViewHolder,
            target: ViewHolder
        ): Boolean {
            helper.rowMoved(viewHolder.adapterPosition, target.adapterPosition)
            return true
        }

        override fun onSwiped(viewHolder: ViewHolder, direction: Int) {

            if (direction == ItemTouchHelper.LEFT)
                (viewHolder as? CustomViewHolder)?.let { helper.rowClear(it) }
        }

        override fun onChildDraw(
            c: Canvas,
            recyclerView: RecyclerView,
            viewHolder: ViewHolder,
            dX: Float,
            dY: Float,
            actionState: Int,
            isCurrentlyActive: Boolean
        ) {
            // dX, kullanıcının yatay eksendeki kaydırma miktarını temsil eder
            // isCurrentlyActive, kullanıcı öğeyi sürüklediği veya kaydırdığı durumu belirtir

            if (actionState == ItemTouchHelper.ACTION_STATE_SWIPE) {
                // Swipe işlemi sırasında çizim yap

                // Sağ tarafta bir silme simgesi çiz
                val icon = ContextCompat.getDrawable(context, R.drawable.ic_delete)
                val iconMargin = (viewHolder.itemView.height - icon?.intrinsicHeight!!) / 2

                // Sağa kaydırma durumunda
                if (dX > 0) {
                    icon.setBounds(
                        viewHolder.itemView.left + iconMargin,
                        viewHolder.itemView.top + iconMargin,
                        viewHolder.itemView.left + iconMargin + icon.intrinsicWidth,
                        viewHolder.itemView.bottom - iconMargin
                    )
                }
                // Sola kaydırma durumunda
                else {
                    icon.setBounds(
                        viewHolder.itemView.right - iconMargin - icon.intrinsicWidth,
                        viewHolder.itemView.top + iconMargin,
                        viewHolder.itemView.right - iconMargin,
                        viewHolder.itemView.bottom - iconMargin
                    )
                }

                // İkona gölge ekleyebilirsiniz
                icon.alpha = (kotlin.math.abs(dX / recyclerView.width)).times(255).toInt()

                icon.draw(c)
            }

            // Diğer çizim işlemlerini gerçekleştir
            super.onChildDraw(c, recyclerView, viewHolder, dX, dY, actionState, isCurrentlyActive)
        }
    }

}

```


## Örnek Kullanım
```
class OrderRecyclerView
@JvmOverloads constructor(
    context: Context,
    attributeSet: AttributeSet? = null,
    style: Int = 0
) : BaseRecyclerView<Order>(context, attributeSet, style) {

    override val itemLayoutId = R.layout.row_order
    override val currentList = arrayListOf<Order>()

    private var onChangeCallback: (Order?) -> Unit = {}
    fun addChangeListener(listener: (Order?) -> Unit) {
        onChangeCallback = listener
    }

    private var onRemoveCallback: (Order?) -> Unit = {}
    fun addRemoveListener(listener: (Order?) -> Unit) {
        onRemoveCallback = listener
    }

    override fun bind(viewHolderBinding: ViewDataBinding, item: Order) {
        (viewHolderBinding as? RowOrderBinding)?.apply{
            model = item
        }
    }

    override fun onRowClear(viewHolder: CustomViewHolder) {
        super.onRowClear(viewHolder)
    }

    companion object{

        @SuppressLint("NotifyDataSetChanged")
        @BindingAdapter("loadData")
        @JvmStatic
        fun loadData(view: OrderRecyclerView, list: OrderList?){
            if (list != null) {
                view.currentList.clear()
                view.currentList.addAll(list)
                view.adapter?.notifyDataSetChanged()
            }
        }
    }
}

```
