---
layout: post
title:  "管件设计"
date:   2019-09-20
excerpt: "对于客户提出增加产品类型需求的设计"
tag:
- pipe 
- design
comments: false
---

起初，客户提出只有直管产品，所以设计少了一些，后期提出管件、钢套钢等，每一种的校验逻辑和计算物料用量方法都不一样。
所以就稍微重新设计了产品这一块，以适应本期要求，及后期再加产品（事实上后期确实又加了三种产品分类）

代码基于springMVC，使用反射注入了pipe类中spring管理的的bean


首先设计了 pipe模板类，用的时候直接调用getHBPlan()即可，会调用各种方法来获取黑白料计划。

``` java
public  abstract class PipeTemplate extends InitNewService {
    /**
     * 获取黑白料计划
     * @Author gaoyuan
     * @Date 2019/1/28 16:46
     * @Param []
     * @return java.util.List<com.penghai.imes.api.productplan.follow.dto.ProductOrderSplitReceiptDetails>
     **/
    public List<ProductOrderSplitReceiptDetails> getHBPlan() {
        if (!preCheck()) return null;
        setProduct();
        calculateCosts();
        return getReceiptDetails();
    }

    public boolean check() {
        checkBeforeOnline();
        return false;
    }
    // 前置检查 包含三项数据合法性验证
    abstract boolean preCheck();
    // 检查订单信息
    abstract boolean checkOrder();
    // 上线校验方法
    abstract boolean checkBeforeOnline();
    abstract void setProduct();
    abstract void calculateCosts();
    abstract List<ProductOrderSplitReceiptDetails> getReceiptDetails();
}


```

产品分类接口，有计算长度，物料用量及上线校验三个接口，事实上，后面只用到了前两个接口
``` java
public interface ProductType {
    /**
     * 计算该领料单算黑白料的长度
     * @Author gaoyuan
     * @Date 2019/1/28 16:57
     * @Param [receiptNo]
     * @return java.math.BigDecimal
     *
     * @param materialIns
     * */
    BigDecimal calculateLength(List<ProductOrderSplitMaterialIn> materialIns, String inside, String outside);

    /**
     * 上线校验接口
     * @Author gaoyuan
     * @Date 2019/1/28 16:57
     * @Param [receipt]
     * @return boolean
     *
     * @param materialIns
     * */
    String[] check();

    /**
     * 根据条件计算黑白料
     * @Author gaoyuan
     * @Date 2019/1/28 17:01
     * @Param [length, inside, outside, order]
     * @return java.math.BigDecimal[]
     **/
    BigDecimal[] calculateCosts(BigDecimal length, BaseMaterialBaseInfo inside, BaseMaterialBaseInfo outside, ProductOrder order);
}
```

管子工厂： 返回管子类型，不同管子类型有不同的计算逻辑
``` java
public class PipeFactory {

    public void getProductType(String orderClassify) {
        ProductType product;
        if (orderClassify == null || "".equals(orderClassify)) {
            // 默认使用塑套钢处理
            product = new STGProduct();
        } else if (ImesConstant.ORDER_CLASSIFY_STEELANDSTEEL.contains(orderClassify)) {
            product = new GTGProduct();
        } else if (ImesConstant.ORDER_CLASSIFY_STEELANDSTEELWH.contains(orderClassify)) {
            // 钢套钢外滑
            product = new GTGWHProduct();
        } else if (ImesConstant.ORDER_CLASSIFY_GLASSANDSTEEL.contains(orderClassify)) {
            // 玻璃钢
            product = new BLGProduct();
        } else if (ImesConstant.ORDER_CLASSIFY_PIPE.contains(orderClassify)) {
            // 管件
            product = new GJProduct();
        } else {
            // 默认塑套钢
            product = new STGProduct();
        }
    }
}
```

管子类，统一封装了模板里面的各种方法，方便调用。
``` java
public class Pipe extends PipeTemplate {
    // 产品类型，不同的产品类型有不同计算黑白料的方式，以及校验所领材料的方式。
    private ProductType product;
    // 日生产任务号/领料单号
    private String receiptNo;
    private ProductOrder order;
    // 所需物料列表
    private List<ProductOrderBom> boms;
    // 物料详情列表（单位物料为一条）
    private List<ProductOrderSplitMaterialIn> materialIns;
    // 黑白料用量
    private BigDecimal whiteCost;
    private BigDecimal blackCost;

    @Autowired
    private ProductOrderSplitReceiptMapper receiptMapper;
    @Autowired
    private ProductOrderSplitMaterialInMapper materialInMapper;
    @Autowired
    private IProductOrderSplitService splitService;
    @Autowired
    private PipeFactory pipeFactory;

    private  Pipe() {
    }

    public Pipe(String receiptNo) {
        this.receiptNo = receiptNo;
    }

    public Pipe(String receiptNo, List<ProductOrderSplitMaterialIn> materialIns) {
        this.receiptNo = receiptNo;
        this.materialIns = materialIns;
    }


    public boolean preCheck() {
//        checkFunction.check(product.checkArray());
        if (!checkOrder()) return false;
        if (!checkReceiptPlan()) return false;
        if (!checkMaterialIn()) return false;
        return true;
    }

    private boolean checkMaterialIn() {
        // 根据领料单号查出实际领用物料信息。
        ProductOrderSplitReceipt receipt = new ProductOrderSplitReceipt();
        receipt.setReceiptNo(receiptNo);
        List<ProductOrderSplitMaterialIn> materialIns = materialInMapper.selectByCondition(receipt);
        if (materialIns == null || materialIns.isEmpty()) {
            return false;
        }
        this.materialIns = materialIns;
        return true;
    }

    private boolean checkReceiptPlan() {
        // 查出领料计划中物料信息（过滤出钢管 pe管）
        ProductOrderSplitReceipt receiptRecord = receiptMapper.selectByReceiptNo(receiptNo);
        List<ProductOrderSplitReceiptDetails> rrDetails = receiptRecord.getDetails();
        if (rrDetails == null || rrDetails.isEmpty()) {
            return false;
        }
        return true;
    }

    public boolean checkOrder() {
        // 根据领料单号查出订单信息，只计算塑套钢产品的黑白料领料计划，对于返修领料，不计算黑白料领料计划
        ProductOrder order = this.receiptMapper.selectOrderInfoByReceiptNo(receiptNo);
        if (order == null) {
           return false;
        }
        this.order = order;
        return true;
    }

    public boolean checkBeforeOnline() {
        ProductOrderSplitReceipt receipt = new ProductOrderSplitReceipt();
        receipt.setReceiptNo(receiptNo);
        receipt.setMaterialList(materialIns);
        // 查出这些物料的所有信息，包括物料代码、类型(这里校验了领料单号和物料的关系)。
        materialIns = materialInMapper.selectByCondition(receipt);
        if (materialIns == null || materialIns.isEmpty()) return false;
        return this.product.check(materialIns);
    }

    public void setProduct() {
        this.product = pipeFactory.getPipe(order.getOrderClassify());
    }

    public void calculateCosts() {
        // set whiteCost and blackCost
        // 通过订单查出bom，可得出内外管信息,
        List<ProductOrderBom> bomList = splitService.getOrderBom(order.getOrderCode());
        this.boms = bomList;
        BaseMaterialBaseInfo inside = null;
        BaseMaterialBaseInfo outside = null;
        for (ProductOrderBom bom : bomList) {
            if (bom.getInside()) {
                inside = receiptMapper.selectMaterialByMaterialCode(bom.getMaterielCode());
                continue;
            } else if (ImesConstant.MATERIAL_TYPE_NEED_CAL_COST.contains(bom.getMaterialType())) {
                outside = receiptMapper.selectMaterialByMaterialCode(bom.getMaterielCode());
                continue;
            }
        }
        if (GTGProduct.class.isInstance(product)) {
            // 2019/5/11 暂这样修改钢套钢的计算逻辑，若为钢套钢需要计算黑白料的种类，则内管外径需加上辅料的厚度。
            // 这里只计算已经领料的辅料，因为目前做bom时会有些物料需求数量为0，即为不需要的物料，不能添加到计算公式中
            Set<String> codeSet = new HashSet<>();
            for (ProductOrderSplitMaterialIn materialIn: materialIns) {
                codeSet.add(materialIn.getMaterialCode());
            }
            for (ProductOrderBom bom : bomList) {
                if (ImesConstant.MATERIAL_TYPE_FL.equals(bom.getMaterialType()) && codeSet.contains(bom.getMaterielCode())) {
                    BaseMaterialBaseInfo fl = receiptMapper.selectMaterialByMaterialCode(bom.getMaterielCode());
                    // 将辅料厚度的两倍添加到外径中
                    BigDecimal toAdd = new BigDecimal(fl.getThickness() == null ? "0" : fl.getThickness()).multiply(BigDecimal.valueOf(2));
                    inside.setOutsideDiameter(new BigDecimal(inside.getOutsideDiameter()).add(toAdd).toString());
                }
            }
        }
        BigDecimal length = product.calculateLength(materialIns, inside.getMaterialCode(), outside.getMaterialCode());
        BigDecimal[] costs = product.calculateCosts(length, inside, outside, order);
        this.whiteCost = costs[0];
        this.blackCost = costs[1];

    }

    public List<ProductOrderSplitReceiptDetails> getReceiptDetails() {
        ProductOrderSplitReceiptDetails white = new ProductOrderSplitReceiptDetails();
        ProductOrderSplitReceiptDetails black = new ProductOrderSplitReceiptDetails();

        // 拼黑白料领料实际
        List<ProductOrderSplitReceiptDetails> details = new ArrayList<>();
        for (ProductOrderBom material: boms) {
            if (ImesConstant.MATERIAL_TYPE_BLACK.equals(material.getMaterialType())) {
                black.setMaterialCode(material.getMaterielCode());
                black.setMaterialType(material.getMaterialType());
                black.setPlanNum(blackCost);
                black.setMeasureUnit(material.getMeasureUnit());
            } else if (ImesConstant.MATERIAL_TYPE_WHITE.equals(material.getMaterialType())) {
                white.setMaterialCode(material.getMaterielCode());
                white.setMaterialType(material.getMaterialType());
                white.setPlanNum(whiteCost);
                white.setMeasureUnit(material.getMeasureUnit());
            }
        }
        details.add(white);
        details.add(black);
        return details;
    }
}
```

具体的产品类，不同的产品类计算长度、损耗和校验的方法不一样。
``` java
public class BLGProduct implements ProductType {

    @Override
    public BigDecimal calculateLength(List<ProductOrderSplitMaterialIn> materialIns, String inside, String outside) {
        BigDecimal length = BigDecimal.ZERO;
        // 玻璃钢计算长度，每根钢管减30cm
        for (ProductOrderSplitMaterialIn materialIn: materialIns) {
            if (inside.equals(materialIn.getMaterialCode())) {
                length = length.add(materialIn.getQuantity().subtract(new BigDecimal(0.3)));
            }
        }
        return length;
    }

    @Override
    public boolean check(List<ProductOrderSplitMaterialIn> materialIns) {
        return false;
    }

    @Override
    public BigDecimal[] calculateCosts(BigDecimal length, BaseMaterialBaseInfo inside, BaseMaterialBaseInfo outside, ProductOrder order) {
        // 根据物料类型：pe管 过滤物料
        // 算出长度，乘以面积，乘以比例，乘以密度。(从订单中获取注料标准信息，其中有比例和密度)
        // 取得比例, 默认黑白料比例格式为白:黑
        BigDecimal[] scales = new BigDecimal[2];
        String[] proportions = order.getProportionName().split(":");
        scales[0] = new BigDecimal(proportions[0]);
        scales[1] = new BigDecimal(proportions[1]);
        // 默认黑白料比例格式为黑:白
        // 三个公式：
        BigDecimal InsulationThickness = new BigDecimal(outside.getOutsideDiameter())
                .subtract(new BigDecimal(inside.getOutsideDiameter())).divide(BigDecimal.valueOf(2), 4, BigDecimal.ROUND_HALF_UP)
                .subtract(new BigDecimal(outside.getThickness()) );
        BigDecimal whiteCost = new BigDecimal(inside.getOutsideDiameter()).add(InsulationThickness)
                .multiply(InsulationThickness).multiply(new BigDecimal(Math.PI)).multiply(order.getDensity())
                .multiply(length).divide(scales[0].add(scales[1]), 4, BigDecimal.ROUND_HALF_UP)
                .divide(new BigDecimal(1000*1000), 4, BigDecimal.ROUND_CEILING);
        BigDecimal blackCost = whiteCost.multiply(scales[1]);
        scales[0] = whiteCost;
        scales[1] = blackCost;
        return scales;
    }
}
```