# abc@using InventoryViewModel = ABW.Application.ViewModel.InventoryViewModel
@model  Dictionary<int, List<InventoryViewModel>>
@using ABW.Application.Helpers;

@{
    Layout = "~/Views/Shared/_DealerLayout.cshtml";
    ViewBag.Title = "Current Inventory";

    ViewBag.Heading = "Current Inventory";
    var tabindex = 0;
    Dictionary<int, int> inventoryAmount = new Dictionary<int, int>();
    if (TempData["inventoryAmount"] != null)
    {
        inventoryAmount = TempData["inventoryAmount"] as Dictionary<int, int>;
    }

    IHtmlString x = null;
    if (TempData["Redirection"] != null)
    {
        x = @Html.Raw(TempData["data"]);
    }
    int bidEventId = 0;
    var agentId = 0;
    var dealerId = 0;
    var countChecker = 0;
    var runTime = "";
    var watchItemCounter = 0;
    if (Model.Count() != 0)
    {
        var inventoryList = Model.Select(i => i.Value).FirstOrDefault();
        bidEventId = inventoryList.Select(i => i.BidEventId).FirstOrDefault();
        agentId = inventoryList.Select(i => i.BidderId).FirstOrDefault();
        dealerId = @ViewBag.DealerId;
        countChecker = inventoryList.Count;
        runTime = @ViewBag.RunTime;
    }
    runTime = @ViewBag.RunTime;
    watchItemCounter = @ViewBag.WatchItemCounter;
}

@section styles{
    <style type="text/css">
        #updateBidsModal .modal-dialog {
            width: auto;
        }

            #updateBidsModal .modal-dialog .table-responsive {
                overflow-y: auto;
            }

        div.dataTables_info {
            text-align: center;
            font-weight: bold;
            font-size: 18px;
        }

        .ie8 .widget .table-bordered td {
            border: #ddd 1px solid;
        }

        table tr .comments, table td .comments {
            word-wrap: break-word;
            max-width: 180px;
        }
    </style>
}
@*Logic to add jquery plugin for showing runtime*@


<script src="~/Scripts/jquery-1.10.2.min.js"></script>
<script src="~/Scripts/jquery.signalR-2.2.0.min.js"></script>
<script type="text/javascript" src="@Url.Content("~/signalr/hubs")"></script>
<script src="~/Scripts/jquery.countdown.js"></script>
<script src='https://cdn.rawgit.com/admsev/jquery-play-sound/master/jquery.playSound.js'></script>

<script>
    jQuery(function ($) {

        $('.hideColumn').attr("style","display:none");
        //startTime();
       

        //Below method is for running timer

        $("td>label[id=txtRunTime]").each(function () {

            var clientDateTime=new Date();
       
            var minsecToAdd=$(this).siblings().val();
        
            var sepMinSec=minsecToAdd.split('  ');
            var hrs=clientDateTime.getHours();
       
            var mins=clientDateTime.getMinutes()+Number(sepMinSec[0]);
            var secs=clientDateTime.getSeconds()+Number(sepMinSec[1]);
         
            if(mins>=60)
            {
                mins=mins-60;
                hrs=hrs+1;
            }
            if(secs>=60)
            {
                secs=secs-60;
                mins=mins+1;
            }
            if(hrs>12)
            {
                hrs=hrs-12;
            }
           
            var formatCountDownString = pad(clientDateTime.getMonth()+1)+"/"+pad(clientDateTime.getDate())+"/"+clientDateTime.getFullYear() + ', ' + hrs + ':' + mins + ':' + secs + ' ' + (clientDateTime.getHours() > 11 ? 'PM' : 'AM');
           
           // alert(clientDateTime.toLocaleString());
            $(this).countdown(formatCountDownString, function (event) {
                $(this).text(event.strftime('%M:%S'));
            }).on('finish.countdown', function () {

                $(this).parents().next().find('button[type=button]').first().attr("disabled", "true");
                $(this).parents().next().find('input[id=Amount]').first().attr("disabled", "true");
                $(this).parents().parents().children('td').filter(':eq(12)').find('input[id=item_MaxBid]').attr("disabled", "true");
            });
        });

        //Will apply colors after page loads
        applyColors();

        //check for disabling controls when user clicks on refresh button.
        // will remove later

        $('tbody#bid-inventory').children('tr').each(function()
        {

            if($(this).children('td').filter(':eq(10)').find('#txtRunTime').text().trim()=="00:00")
            {
                $(this).children('td').filter(':eq(11)').find('input[id=Amount]').attr("disabled", "true");
                $(this).children('td').filter(':eq(12)').find('input[id=item_MaxBid]').attr("disabled", "true");

            }

        });

        if(@watchItemCounter==0)
        {
            $('a#btnBids').removeClass('hide');
        }
       

        //starting the hub here and calling hub method send
        var chat = $.connection.biddingHub;
        
        $.connection.hub.start().done(function () {
            // Call the Send method on the hub.
            var data="";
            $('input[id=hdnInventoryID]').each(function()
            {
                data=data+$(this).val() +";"
            });
            chat.server.createsubgroup(data);    
            chat.server.send(@x);
                
            
            
        });

        //Recieve data from hub here
        chat.client.broadcastMessage = function (p) {


            var amount=0;
            var inventoryID=0;
            var bidEventID=0;
            var bidderID=0;
            var dealerCode="";
            var hdnRunTime="";
            var textRunTime="";
            var columnCount=14;

            //looping through recieved object from hub
            for(var i=0;i<p.length;i++){
                var obj=p[i];
                for(var key in obj){

                    if(obj.hasOwnProperty(key))
                    {
                        if(key=="Amount")
                        {
                            amount=obj[key];
                        }
                        if(key=="InventoryID")
                        {
                            inventoryID=obj[key];
                        }
                        if(key=="BidEventID")
                        {
                            bidEventID=obj[key];
                        }
                        if(key=="BidderID")
                        {
                            bidderID=obj[key];
                        }
                        if(key=="DealerCode")
                        {
                            dealerCode=obj[key];
                        }
                        if(key=="HdnRunTime")
                        {
                            hdnRunTime=obj[key];
                        }
                        if(key=="TextRunTime")
                        {
                            textRunTime=obj[key];
                        }

                    }
                }


                //logic to read incoming values from hub and process it
                $('tbody#bid-inventory').children('tr').each(function()
                {
                   
                    if ($(this).children('td:nth-child(2)').find('input[type=hidden]').val()==inventoryID && $(this).children('td:nth-child(1)').find('input[type=hidden]').val()==bidEventID)
                    {

                        if($(this).children('td:last').find('b#cha').text()<amount)
                        {
                           
                            $.playSound("http://www.noiseaddicts.com/samples_1w72b820/3724");

                            $(this).children('td:last').find('b#cha').text(amount);
                            $(this).children('td:last').find('b#dc').text(dealerCode);
                           
                            if($(this).children('td').length==columnCount)
                            {
                               
                                $(this).children('td').filter(':eq(10)').html('<label id="txtRunTime" style="color:black;">'+textRunTime+'</label><input type="hidden" id="hdnRunTime" value="" />');
                             
                                $(this).children('td').filter(':eq(10)').find('#hdnRunTime').val(hdnRunTime);
                                var clientDateTime=new Date();
         
                                var minsecToAdd=$(this).children('td').filter(':eq(10)').children('#hdnRunTime').val();
                                var sepMinSec=minsecToAdd.split('  ');
                                var hrs=clientDateTime.getHours();
                                var mins=clientDateTime.getMinutes()+Number(sepMinSec[0]);
                                var secs=clientDateTime.getSeconds()+Number(sepMinSec[1]);
                                if(mins>=60)
                                {
                                    mins=mins-60;
                                    hrs=hrs+1;
                                }
                                if(secs>=60)
                                {
                                    secs=secs-60;
                                    mins=mins+1;
                                }
                                if(hrs>12)
                                {
                                    hrs=hrs-12;
                                }
                               
                                var formatCountDownString = pad(clientDateTime.getMonth()+1)+"/"+pad(clientDateTime.getDate())+"/"+clientDateTime.getFullYear() + ', ' + hrs + ':' + mins + ':' + secs + ' ' + (clientDateTime.getHours() > 11 ? 'PM' : 'AM');
                                $(this).children('td').filter(':eq(10)').children('label[id=txtRunTime]').countdown(formatCountDownString, function (s) {

                                    $(this).text(s.strftime('%M:%S'));
                                }).on('finish.countdown', function () {

                                    $(this).parents().next().find('button[type=button]').first().attr("disabled", "true");
                                    $(this).parents().next().find('input[id=Amount]').first().attr("disabled", "true");
                                    $(this).parents().parents().children('td').filter(':eq(12)').find('input[id=item_MaxBid]').attr("disabled", "true");
                                });

                            }
                        }
                    }
                    if ($(this).children('td:nth-child(2)').find('input[type=hidden]').val()==inventoryID && $(this).children('td:nth-child(1)').find('input[type=hidden]').val()==bidEventID && bidderID==@agentId)
                    {
                        if($(this).children('td').length==columnCount)
                        {
                            $(this).children('td').filter(':eq(11)').find('input[id=Amount]').val(amount);
                        }
                        else
                        {
                            $(this).children('td').filter(':eq(10)').find('input[id=Amount]').val(amount);
                        }


                    }

                });
            }

            //apply colors again on basics higger bids
            applyColors();

        };


    //function startTime() {
    //    var today = new Date();
    //    var h = today.getHours();
    //    var m = today.getMinutes();
    //    var s = today.getSeconds();
    //    m = checkTime(m);
    //    s = checkTime(s);
    //    document.getElementById('txt').innerHTML =
    //    h + ":" + m + ":" + s;
    //    var t = setTimeout(startTime, 500);
    //}
    //function checkTime(i) {
    //    if (i < 10) {i = "0" + i};  // add zero in front of numbers < 10
    //    return i;
    //}
    function pad(n) {return n < 10 ? "0"+n : n;}
   
        function applyColors()
        {

            $('tbody#bid-inventory').children('tr').each(function()
            {
                if($(this).children('td').length==14){
                    var amountValue=$(this).children('td').filter(':eq(11)').find('input[id=Amount]').val().trim();
                }
                else{
                    var amountValue=$(this).children('td').filter(':eq(10)').find('input[id=Amount]').val().trim();
                }
                if($(this).children('td:last').find('b#cha').text().trim()== amountValue)
                {

                    $(this).addClass("statusSold");
                    $(this).removeClass("statusPull");
                }
                else{
                 
                    if(amountValue !=0)
                    {

                        $(this).addClass("statusPull");
                        $(this).removeClass("statusSold");
                    }
                }
            });
        }

        var time = new Date().getTime();
        $(document.body).bind("mousemove keypress", function(e) {
            time = new Date().getTime();
        });
        //setTimeout(refresh, 60000);

        var inventoryID= 0;
        var agent=@agentId;
        var bideventID=@bidEventId;
        //once page gets loaded lets check for the view open/closed eye logic
        $.ajax({
            url: '@Url.Action("UpdateBidWatch", "BidEvent")',
            type: "POST",
            data: JSON.stringify({ 'Id': inventoryID,'agentId':agent,'bideventID':bideventID }),
            datatype: "json",
            contentType: "application/json; charset=utf-8",
            success: function (data) {
                if(data>0)
                {
                    $("#chkWatched").show();
                }
                else{
                    $("#chkWatched").hide();
                }

            },
            error: function (data) {

            }
        });





        $('td>.isWatched').each(function () {
            $(this).click(function () {

                if ($(this).find('img').attr('id') == "closedeye") {
                    $(this).find('img').removeAttr('src');
                    $(this).find('img').attr('src', '/Tenants/Default/img/openeye.gif');
                    $(this).find('img').attr('id','openeye');
                    $(this).find('img').siblings().text('Watching');
                    $(this).removeAttr('style');

                }
                else {
                    $(this).find('img').removeAttr('src');
                    $(this).find('img').attr('src', '/Tenants/Default/img/closedeye.gif');
                    $(this).find('img').attr('id','closedeye');
                    $(this).find('img').siblings().text('Watch Run');
                    $(this).css("background-color","yellow");
                }

                var inventoryID= $(this).siblings().val();
                var agent=@agentId;
                var bideventID=@bidEventId;


                $.ajax({
                    url: '@Url.Action("UpdateBidWatch", "BidEvent")',
                    type: "POST",
                    data: JSON.stringify({ 'Id': inventoryID,'agentId':agent,'bideventID':bideventID }),
                    datatype: "json",
                    contentType: "application/json; charset=utf-8",
                    success: function (data) {
                        if(data>0)
                        {
                            $("#chkWatched").show();
                        }
                        else{
                            $("#chkWatched").hide();
                        }
                        chat.server.removesubgroup(inventoryID);
                    },
                    error: function (data) {

                    }
                });

            });

        });

        $('#chkWatched').click(function()
        {

            var ischecked=$(this).text();
            $.ajax({
                url: '@Url.Action("Bids", "BidEvent")',
                type: "POST",
                data: JSON.stringify({ 'watched': ischecked.trim() }),
                datatype: "json",
                contentType: "application/json; charset=utf-8",
                success: function (data) {
                    if(ischecked.trim()=="View All")
                    {
                        

                        location.reload();
                    }
                    else{
                        $('div#topHeader').html(data);
                        $('div.page-head:last').hide();
                        $('div.page-head').prepend("<div style='height:40px;'></div>")
                        $('header').hide();
                      
                        $('#chkWatched').text('View All');
                        $('div.matter').removeAttr('class');
                        $('div.widget').removeAttr('widget');
                        $('footer:first').hide();
                        var data="";
                        $('input[id=hdnInventoryID]').each(function()
                        {
                            data=data+$(this).val() +";"
                        });
                        chat.server.createsubgroup(data);
                    }
                },
                error: function (data) {

                }
            });
        });

    $('.isWatched').each(function()
    {
          
        $(this).find('img[id=closedeye]').parent().css("background-color","yellow");
          
    });
    




    });





    function refresh() {

        if(new Date().getTime() - time >= 60000)
            window.location.reload(true);

    }
                

</script>

    <div class="row" id="topHeader">
    <!-- File Upload widget -->
        <div class="row">
            <div class="col-md-offset-4">
                <b style="font-size:large">Click "Update Watched Items" to view all inventory and Watch more runs.</b>
            </div>
            
            
        </div>
        @*<div class="row">
            <div class=" col-md-offset-1 col-md-9"><b>If your system time does not match the time shown towards the right, your bid countdown timer may be off based on your system time. Do not wait until the last minute to place bids as they may be rejected.</b></div>
            <b style="font-size:large">System Time:</b>
            <b id="txt" class="cold-md-2" style="font-size:large"></b>
        </div>*@
    <div class="col-md-12">
        <div class="widget">
            <!-- Widget title -->
            <div class="widget-head">
                <div class="pull-left">Bids</div>
                <div class="widget-icons pull-right">
                    <a href="#" class="wminimize"><i class="icon-chevron-up"></i></a>
                </div>
                <div class="clearfix"></div>
            </div>
            <div class="widget-content referrer ">
                <div class="padd">
                    <div class="pull-right">
                        <a href="@Url.Action("WatchBids", "BidEvent")" id="btnBids" class="btn btn-primary hide">Update Watched Items</a>
                    </div>
                    @foreach (KeyValuePair<int, List<InventoryViewModel>> entry in Model)
{
    if (entry.Value.Count > 0)
    {
        var inventoryViewList = entry.Value;

        var inventoryView = inventoryViewList.FirstOrDefault();
        var hasPreview = inventoryView.HasPreview;
        var isFloorPriceVisible = inventoryView.IsFloorPriceVisible;
        <div class="panel panel-default">
            <div class="panel-heading">
                <div class="col-md-3">
                    <h3>
                        @inventoryView.Title
                    </h3>
                </div>
                <div class="pull-right">
                    <a href="@Url.Action("WatchBids", "BidEvent")" id="btnBids" class="btn btn-primary">Update Watched Items</a>
                    <a href="@Url.Action("PrintInventory", new { Id = inventoryView.BidEventId })" id="btnPrintInventory" class="btn btn-primary" target="_blank">Print Inventory</a>
                    <a href="@Url.Action("PrintBids", new { Id = inventoryView.BidEventId })" id="btnPrintBids" class="btn btn-primary" target="_blank">Print Bids</a>
                    @if (inventoryView.StartDate <= ViewBag.ServerUtcTime)
                    {
                        <input type="button" value="Submit Bids" class="btn btn-primary update btn-success" tabindex="@tabindex" />
                    }
                </div>
                <div class="clearfix"></div>
            </div>

            <div class="clearfix"></div>

            <div class="mobileSortContainer">
                <strong>Sort by: </strong>
                <select id="mobileSort">
                    <option value="">- Choose Column -</option>
                    <option value="0">@Html.DisplayNameFor(i => inventoryView.RunNo)</option>
                    <option value="1">@Html.DisplayNameFor(i => inventoryView.IsWatched)</option>
                    <option value="2">@Html.DisplayNameFor(i => inventoryView.Seller)</option>
                    <option value="3">@Html.DisplayNameFor(i => inventoryView.YearMakeModel)</option>
                    <option value="4">@Html.DisplayNameFor(i => inventoryView.Color)</option>
                    <option value="5">@Html.DisplayNameFor(i => inventoryView.Mileage)</option>
                    <option value="6">@Html.DisplayNameFor(i => inventoryView.TitleStatus)</option>
                    <option value="7">@Html.DisplayNameFor(i => inventoryView.Description)</option>
                    <option value="8">@Html.DisplayNameFor(i => inventoryView.VIN)</option>
                    <option value="9">@Html.DisplayNameFor(i => inventoryView.MarketValue)</option>
                    @{
                                        var fpPos = 10;
                                        var tsPos = 10;
                                        var baPos = 10;
                                        var mbPos = 10;
                                        var cbPos = 11;
                                        if (isFloorPriceVisible)
                                        {
                                            tsPos = tsPos + 1;
                                            baPos = baPos + 1;
                                            mbPos = mbPos + 1;
                                            cbPos = cbPos + 1;
                                        }
                                        if (ViewBag.TimerStart)
                                        {
                                            baPos = baPos + 1;
                                            mbPos = mbPos + 1;
                                            cbPos = cbPos + 1;
                                        }
                                        if (ViewBag.CanAgentBid && inventoryView.StartDate <= ViewBag.ServerUtcTime)
                                        {
                                            mbPos = mbPos + 1;
                                            cbPos = cbPos + 1;
                                        }
                    }
                    @if (isFloorPriceVisible)
                                    {
                                    <option value="@fpPos">FloorPrice</option>
                                    }
                    @if (ViewBag.TimerStart)
                                    {
                                    <option value="@tsPos">@Html.DisplayNameFor(i => inventoryView.RunTime)</option>
                                    }
                    @if (ViewBag.CanAgentBid && inventoryView.StartDate <= ViewBag.ServerUtcTime)
                                    {
                                    <option value="@baPos">@Html.DisplayNameFor(i => inventoryView.Bid.Amount)</option>
                                    }
                    <option value="@mbPos">Max Bid</option>
                    <option value="@cbPos">Current Highest Bid</option>
                </select>
                <div id="orderControl">
                    <input id="radio-asc" type="radio" name="order" value="asc" />ASC
                    <input id="radio-desc" type="radio" name="order" value="desc" />DESC
                </div>
            </div>
            <div class="table-responsive bidForms">
                <table class="table table-striped table-bordered table-hover bidTable" id="bidTable@(inventoryView.BidEventId)">
                    <thead>
                        <tr>
                            <th>
                                @Html.DisplayNameFor(i => inventoryView.RunNo)
                            </th>
                            <th>
                                @Html.DisplayNameFor(i => inventoryView.IsWatched)
                            </th>
                            <th>
                                @Html.DisplayNameFor(i => inventoryView.Seller)
                            </th>
                            <th>
                                @Html.DisplayNameFor(i => inventoryView.YearMakeModel)
                            </th>
                            <th>
                                @Html.DisplayNameFor(i => inventoryView.Color)
                            </th>
                            <th>
                                @Html.DisplayNameFor(i => inventoryView.Mileage)
                            </th>
                            <th>
                                @Html.DisplayNameFor(i => inventoryView.TitleStatus)
                            </th>
                            <th>
                                @Html.DisplayNameFor(i => inventoryView.Description)
                            </th>
                            <th>
                                @Html.DisplayNameFor(i => inventoryView.VIN)
                            </th>
                            <th class="hideColumn">
                                @Html.DisplayNameFor(i => inventoryView.MarketValue)
                            </th>

                            @if (isFloorPriceVisible)
                                            {
                                            <th>
                                                FloorPrice
                                            </th>
                                            }
                            @if (ViewBag.TimerStart)
                                            {
                                            <th>
                                                @Html.DisplayNameFor(i => inventoryView.RunTime)
                                            </th>
                                            }
                            @if (ViewBag.CanAgentBid && inventoryView.StartDate <= ViewBag.ServerUtcTime)
                                            {
                                            <th class="bid-amount col-md-2">
                                                @Html.DisplayNameFor(i => inventoryView.Bid.Amount)
                                            </th>
                                            }
                            @if (inventoryView.StartDate <= ViewBag.ServerUtcTime)
                            { 
                                <th>
                                    Max Bid
                                </th>
                            }
                            @if (inventoryView.StartDate <= ViewBag.ServerUtcTime)
                            {
                            <th>
                                Current Highest Bid
                            </th>
                            }
                        </tr>
                    </thead>
                    <tbody id="bid-inventory">
                        @foreach (var item in inventoryViewList.OrderBy(i => i.RunNo))
                                        {
                                            string status = "";
                                            ViewBag.tabindex = ++tabindex;
                                        <tr>
                                            <td class="col-lg-1 item-run-number">
                                                <span class="run-number">
                                                    @Html.DisplayFor(modelItem => item.RunNo)
                                                    <input type="hidden" name="bidEventId" id="bidEventId" value="@bidEventId" />
                                                </span>
                                                @if (item.HasImage)
                                                    {
                                                    <i class="icon-camera btn btn-default btn-action" id="@item.VehicleId">
                                                    </i>
                                                    }
                                            </td>
                                            <td class="item-watch-bid">
                                                <a class="btn btn-info btn-group-lg isWatched aligncenter">
                                                    @if (item.IsWatched ?? false)
                                                        {
                                                        <img id="openeye" src="~/Tenants/Default/img/openeye.gif" /><b style="color:black;">Watching</b>
                                                        }
                                                    else
                                                    {
                                                        <img id="closedeye" src="~/Tenants/Default/img/closedeye.gif" /><b style="color:black;">Watch Run</b>
                                                    }
                                                </a>
                                                <input type="hidden" id="hdnInventoryID" value="@item.InventoryId" />
                                            </td>
                                            <td class="dealer-name item-dealer-name">
                                                @Html.DisplayFor(modelItem => item.Seller)
                                            </td>
                                            <td class="year-make-model item-year-make-model">
                                                @Html.DisplayFor(modelItem => item.YearMakeModel)
                                            </td>
                                            <td class="item-color">
                                                @Html.DisplayFor(modelItem => item.Color)
                                            </td>
                                            <td class="item-mileage">
                                                @Html.DisplayFor(modelItem => item.Mileage)
                                            </td>
                                            <td class="item-title">
                                                @Html.DisplayFor(modelItem => item.TitleStatusType)
                                            </td>
                                            <td class="comments item-comments">
                                                @Html.DisplayFor(modelItem => item.Description)
                                            </td>
                                            <td class="item-vin">
                                                @Html.DisplayFor(modelItem => item.VIN)
                                            </td>
                                            <td class="item-market-value hideColumn">
                                                @Html.DisplayFor(modelItem => item.MarketValue)
                                            </td>
                                            @if (isFloorPriceVisible)
                                                {
                                                <td class="floorprice item-floor-price">
                                                    @if (item.FloorPrice > 0)
                                                        {
                                                        @Html.DisplayFor(modelItem => item.FloorPrice)
                                                        }
                                                </td>
                                                }
                                            @if (ViewBag.TimerStart)
                                                {
                                                <td class="item-timer-start">
                                                    @*@Html.DisplayFor(modelItem => item.RunTime,new { htmlAttributes = new { @id = "txtRunTime" } })*@
                                                    <label id="txtRunTime" style="color:black;">@item.RunTime</label>
                                                    <input type="hidden" id="hdnRunTime" value="@item.hdnMinSec" />
                                                </td>


                                                }

                                            @if (ViewBag.CanAgentBid && item.StartDate <= ViewBag.ServerUtcTime)
                                                {

                                                    if (item.Bid == null)
                                                    {
                                                        item.Bid = new ABW.Application.Models.Bid();
                                                        item.Bid.InventoryId = item.InventoryId;
                                                        item.Bid.BidEventId = item.BidEventId;
                                                        item.Bid.BidderId = item.BidderId;
                                                        item.Bid.Amount = item.Amount == null ? 0 : (int)item.Amount;
                                                        item.Bid.DealerId = item.SellerId;

                                                    }
                                                    if (inventoryAmount.ContainsKey(item.InventoryId))
                                                    {
                                                        item.Bid.Amount = inventoryAmount[item.InventoryId];

                                                        status = "form-group has-error";
                                                    }
                                                    item.Bid.RunTime = item.RunTime;
                                                    //TODO: READD THIS
                                                    //item.Bid.DealerId = item.SellerId;
                                                <td class="bid-amount options actions col-md-1 ">
                                                    @using (Ajax.BeginForm("UpdateBid", "Vehicle", item.Bid,
                                                            new AjaxOptions()
                                                            {
                                                                InsertionMode = InsertionMode.Replace,
                                                                UpdateTargetId = "bid-container-" + item.InventoryId,
                                                                LoadingElementId = "loading-" + item.InventoryId,
                                                                OnComplete = "setTabIndex('#bid-container-" + item.InventoryId + "')"
                                                            },
                                                        new
                                                        {
                                                            name = "updateforms",
                                                            @class = "form-inline"
                                                        }))
                                                        {
                                                        <input type="hidden" class="tabindex" value="@tabindex" />
                                                        <input type="hidden" id="maxAmount" />
                                                        <div id="bid-container-@item.InventoryId" class="@status">
                                                            @Html.Partial("~/Views/Vehicle/UpdateBid.cshtml", item.Bid,
                                                                 new ViewDataDictionary {
                                                                     { "isHideBidButton", true },
                                                                     { "isShowDeleteButton", false },
                                                                     {"DealerId",dealerId}


                                                                })

                                                        </div>
                                                        }
                                                </td>
                                                }


                                          @if (inventoryView.StartDate <= ViewBag.ServerUtcTime)
                                             {
                                                <td class="item-max-bid">
                                           
                                                    @if (item.SellerId == dealerId)
                                                        {
                                                        @Html.TextBoxFor(modelItem => item.MaxBid, new { Name = "MaxBid", @class = "MaxBid Amount form-control sortable", @disabled = "true" })
                                                        }
                                                        else
                                                        {
                                                        @Html.TextBoxFor(modelItem => item.MaxBid, new { Name = "MaxBid", @class = "MaxBid Amount form-control sortable" })
                                                        }
                                                    <input type="hidden" class="old-MaxBid" value="item.MaxBid" />
                                                </td>
                                             
                                            <td class="item-current-highest-bid">
                                                @if (item.CurrentHighestBid != null)
                                                { 
                                                <b style="font-size:large;">$</b><b id="cha" style="font-size:large;">@Html.DisplayFor(modelItem => item.CurrentHighestBid)</b><br/> <b id="dc" style="font-size:large;"> @Html.DisplayFor(modelItem => item.DealerCode)</b>
                                                }
                                                else
                                                {
                                                    <b id="cha" style="font-size:large;">@Html.DisplayFor(modelItem => item.CurrentHighestBid)</b><br /> <b id="dc" style="font-size:large;"> @Html.DisplayFor(modelItem => item.DealerCode)</b>
                                                }
                                            </td>
                                          }
                                        </tr>
                                        }
                    </tbody>
                </table>
            </div>
            <div class="panel-footer">
                @if (inventoryView.StartDate <= ViewBag.ServerUtcTime)
                                {
                                <div class="pull-right">
                                    <input type="button" value="Submit Bids" class="btn btn-primary update btn-success" tabindex="@tabindex" />
                                </div>
                                }
                <div class="clearfix"></div>
            </div>
        </div>
    }
}
                    @if (countChecker == 0 && watchItemCounter !=0)
                    {
                        <div style="text-align:center;"><b>@Html.Raw(Resources.BiddingClosedCurrentInventory)@runTime@Html.Raw(Resources.BiddingClosedCurrentInventorySecond)</b></div>

                    }
                    else if (watchItemCounter == 0)
                    {
                        <div style="text-align:center;"><b>@Html.Raw(Resources.UpdateWatchList)</b></div>
                    }
                </div>
            </div>
            <div class="widget-foot">
                <div class="clearfix"></div>
            </div>
        </div>
    </div>


    <div class="modal fade" id="updateBidsModal" tabindex="-1" role="dialog" aria-labelledby="myModalLabel" aria-hidden="true">
        <div class=" modal-dialog">
            <div class="modal-content">
                <div class="modal-header">
                    <button type="button" class="close" data-dismiss="modal" aria-hidden="true">&times;</button>
                    <h4 class="modal-title">Please confirm your bids</h4>
                </div>
                <div class="modal-body">
                    <div>
                        @Html.Raw(Resources.BiddingInventorycount)
                    </div>
                    <div class="clearfix"></div>
                    <div class="table-responsive" id="bidConfirm" style="max-height:500px">

                    </div>
                </div>
                <div class="modal-footer">
                    <div class="clearfix">
                        <a href='#' id="PrintConfirmBids" class="btn btn-primary">Print Bids</a>
                        <input type="button" value="Confirm" id="updateBids" class="btn btn-primary" />
                        <button type="button" class="btn btn-default" data-dismiss="modal">Close</button>
                    </div>
                </div>
            </div>
        </div>
    </div>

    <div class="modal fade" id="vehicleImage" tabindex="-1" role="dialog" aria-labelledby="gallery" aria-hidden="true">
        <div class=" modal-dialog" style="max-width: 432px">
            <div class="modal-content">
            </div>
        </div>
    </div>

     </div>

    @using (Html.BeginForm("UpdateBids", "Vehicle", FormMethod.Post, new { id = "updateBids" })) { }
    @using (Html.BeginForm("PrintConfirmBids", "BidEvent", FormMethod.Post, new { id = "confirmBidsForm", target = "_blank" })) { }
    @section Scripts {
        <script src="~/Scripts/jquery.dataTables.plugins.js"></script>
        <script>

                        function setTabIndex(selector) {
                            $selector = $(selector)
                            $selector.find('.Amount').attr('tabindex', $selector.siblings('.tabindex').val())
                            $selector.find('.Amount').focus();
                        }
                        $(document).ready(function () {

                            if ($('.Amount').length > 0) {
                                $('.Amount')[0].focus();
                            }
                            $('#vehicleImage').on('hidden.bs.modal', function () {
                                $(this).removeData('bs.modal');//bootsrtap3
                            });

                            $('body').on('keyup change', 'input.Amount', function () {
                                if (parseInt($(this).val()) >= 0 && $(this).val() != $(this).siblings('.old-amount').val()) {
                                    $(this).siblings('.amount-modified').val('1');
                                } else {
                                    $(this).siblings('.amount-modified').val('0');
                                }
                            });
                        
                            $('body').on('keyup change', 'input.MaxBid', function () {
                                if (parseInt($(this).val()) >= 0 && $(this).val() != $(this).siblings('.old-MaxBid').val() && parseInt($(this).parent().closest('td').prev('td').find('#Amount').val())>0) {
                                   
                                    $(this).parent().closest('td').prev('td').find('.amount-modified').val('1');
                                } 
                            });
                            $('body').on('focusin', 'input.Amount', function () {
                                if (parseInt($(this).val()) <= 0) {
                                    $(this).val('');
                                }
                            });

                            $('body').on('focusout', 'input.Amount', function () {
                                if (!$(this).val()) {
                                    $(this).val('0');
                                }
                            });

                            $('#vehicleImage').on('loaded.bs.modal', function () {
                                LoadPrettyPhoto();
                            });

                            $(".update").click(function () {

                                if($('#chkWatched').text()=="View All")
                                {
                                    sessionStorage.setItem("previousValue","1");
                                }


                                $('#bidConfirm').html($(this).parents('.panel').find('.bidTable').clone(true));

                                $("#bidConfirm .bidTable input.amount-modified").each(function () {

                                    if ($(this).val() != '1') {
                                        $(this).closest('tr').remove();
                                    }
                                });

                                $('#inventoryCount').text($("#bidConfirm .bidTable tbody tr").length);
                                $('#bidConfirm .bidTable input').attr('disabled', 'disabled');

                                $('#bidConfirm .bidTable .bid-amount button').hide();
                                $('#bidConfirm .bidTable .save').hide();
                                $('#updateBidsModal').modal();

                                $('#bidConfirm').find('a').removeClass('btn-info');

                                $('#PrintConfirmBids').hide();

                                $('input[id=updateBids]').removeClass('btn-primary');
                                $('input[id=updateBids]').addClass('btn-success')

                            });

                            $('body').on('click', '.save', function (event) {
                                event.preventDefault();
                                $this = $(this);
                                var isSubmitOk = confirm("Do you want to post a bid $"
                                    + $this.parents('td').find('.Amount').val()
                                    + " on\n"
                                    + $.trim($this.parents('tr').find('.year-make-model').text())
                                    );
                                if (isSubmitOk) {
                                    $(this).parents('form:first').submit();
                                } else {
                                    $this.siblings('.Amount').val($this.siblings('.old-amount').val());
                                    $this.siblings('.Amount').focus();
                                }
                            });

                            $('body').on('click', '.icon-camera', function (event) {
                                event.preventDefault();
                                var Id = $(this).attr('id');
                                $.ajax({
                                    url: '@Url.Action("ImageURL", "VehicleGallery")',
                                    type: "POST",
                                    data: JSON.stringify({ 'Id': Id }),
                                    datatype: "json",
                                    contentType: "application/json; charset=utf-8",
                                    success: function (data) {
                                        var url = $.parseJSON(data);
                                        $.prettyPhoto.open(url);
                                    },
                                    error: function (data) {
                                    }
                                });
                            });

                            $('body').on('click', '.delete', function (event) {
                                event.preventDefault();
                                $this = $(this);
                                $this.siblings('.Amount').val('0');
                                $this.siblings('.deleteBid').val('true');
                                $(this).parents('form:first').submit();
                            });
                         
                            $('body').on('click', '#updateBids', function () {

                                $('#updateBidsModal').modal('hide');

                                var maxBid=0;
                                var bids = new Array();
                                var data = "";
                                var i = 0;
                                var $form = $('#updateBids');
                                $('.bidForms .bidTable input.amount-modified').each(function () {
                                    if ($(this).val() == '1') {
                                        //logic for pushing maxbid here

                                        if($(this).parent().parent().parent().parent().siblings().length==12)
                                        {
                                            maxBid=$(this).parent().parent().parent().parent().siblings().filter(':eq(10)').find('input').val();
                                        }
                                        else{
                                            maxBid=$(this).parent().parent().parent().parent().siblings().filter(':eq(11)').find('input').val();
                                        }

                                        var item = $(this).closest('form[name="updateforms"]').serializeArray();

                                        for (var b = 0; b < item.length; b++) {

                                            $("<input>").attr({
                                                'type': 'hidden',
                                                'name': '[' + i + '].' + item[b].name
                                            }).val(item[b].value).appendTo($form);
                                        }

                                        $("<input>").attr({
                                            'type': 'hidden',
                                            'name': '[' + i + '].' + 'MaxBid'
                                        }).val(maxBid).appendTo($form);
                                        i++;

                                    }



                                });
                                if (i > 0) {


                                    $form.submit();


                                }
                            });

                            @if (ViewBag.CanAgentBid )
                            {
                                @:var targets = [3, 4, 5, 6, 8];
                            }
                            else
                            {
                                @:var targets = [3, 4, 5, 6];

                            }

                            dataTableRef =  $('.bidTable').dataTable({
                                "paging": false,
                                "dom": 'T<"clear">litipr',
                                "searching": false,
                                "columnDefs": [
                                    @if (ViewBag.CanAgentBid )
                                {
                                @: { "orderDataType": "dom-numeric", "targets": ["bid-amount"], "type": "numeric" },
                                }
                        { "orderable": false, "targets": targets }, { "orderable": true, "targets": [0, 1, 2, 7] }
                        ],
                    "language": {
                        "emptyTable": "No data available at this time. Bidding is closed",
                        "sInfo": '@Resources.BidsPaginationText.',
                        "sInfoEmpty": ""
                    },
                    "fnPreDrawCallback": function (oSettings) {
                        /* reset currData before each draw*/
                        currData = [];
                    },
                    "fnRowCallback": function (nRow, aData, iDisplayIndex, iDisplayIndexFull) {
                        currData.push(aData);

                    },
                    "fnDrawCallback": function (oSettings) {
                        $('.Amount').each(function (i) {
                            $(this).attr('tabindex', i + 1);
                        });
                        if ($('.Amount').length > 0) {
                            $('.Amount')[0].focus();
                        }
                    }
                        });

                        });
                        
           
            var colNum = 0;
            $('#mobileSort').change(function(){
                dataTableRef.fnSort([$(this).val(),'asc']);
                $("#radio-asc").prop("checked", true)
                colNum = $(this).val();
                //Added this focus because I can't read minified code
                $('#mobileSort').focus();
            });
            $('#orderControl').change(function(){
                dataTableRef.fnSort([colNum,$("#orderControl input[type='radio']:checked").val()]);
                //Added this focus because I can't read minified code
                $('#mobileSort').focus();
            });

                        //function printCurrentBids(){
                        $('body').on('click', '#PrintConfirmBids', function (event) {
                            event.preventDefault();
                            var bids = new Array();
                            var data = new Array();
                            var $form = $('#confirmBidsForm');
                            var headers = new Array("RunNo", "Seller", "YearMakeModel", "Color", "Mileage", "TitleStatus", "Comment", "VIN", "MarketValue", "Amount");
                            var i = 0; j = 0; k = 0;
                            $('#bidConfirm table.bidTable tr td').each(function () {
                                if (j == 0)
                                    data[i] = new Array();
                                if (j < headers.length - 1) {
                                    $("<input>").attr({
                                        'type': 'hidden',
                                        'name': '[' + k + '].' + headers[j]
                                    }).val($(this).text().trim()).appendTo($form);
                                }
                                else {
                                    $("<input>").attr({
                                        'type': 'hidden',
                                        'name': '[' + k + '].' + headers[j]
                                    }).val($(this).find('.Amount').val().trim()).appendTo($form);
                                }
                                j++;
                                if (j == headers.length) {
                                    j = 0;
                                    k++
                                }
                            });
                            $("<input>").attr({
                                'type': 'hidden',
                                'name': 'bidEventId'
                            }).val(@bidEventId).appendTo($form);
                            if (k > 0) {
                                $form.submit();
                            }
                            else {
                                alert('Sorry, There are no bid(s) to print.')
                            }
                        });

                        if(sessionStorage.getItem("previousValue")=="1")
                        {

                            $.ajax({
                                url: '@Url.Action("Bids", "BidEvent")',
                                type: "POST",
                                data: JSON.stringify({ 'watched': true }),
                                datatype: "json",
                                contentType: "application/json; charset=utf-8",
                                success: function (data) {

                                    $('div#topHeader').html(data);
                                    $('div.page-head:last').hide();
                                    $('div.page-head').prepend("<div style='height:40px;'></div>")
                                    $('header').hide();
                                    $('#chkWatched').text('View All');
                                    $('div.matter').removeAttr('class');
                                    $('div.widget').removeAttr('widget');
                                    $('footer:first').hide();

                                },
                                error: function (data) {

                                }
                            });

                            sessionStorage.removeItem("previousValue");

                        }


        </script>
    }
