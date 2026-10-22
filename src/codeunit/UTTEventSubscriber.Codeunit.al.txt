codeunit 67023 "UTT EventSubscriber"
{
    [EventSubscriber(ObjectType::Codeunit, Codeunit::"Release Purchase Document", 'OnCodeOnAfterCheck', '', false, false)]
    local procedure OnCodeOnAfterCheck(PurchaseHeader: Record "Purchase Header"; var PurchaseLine: Record "Purchase Line")
    var
        GlobalVariables: Codeunit "KVS Global Variables";
        PurchaseLineLoc: Record "Purchase Line";
    begin
        if not GlobalVariables.IsMexicoCompany() then
            exit;

        if PurchaseHeader."Buy-from Country/Region Code" = 'MX' then
            exit;

        PurchaseLineLoc.Copy(PurchaseLine);
        PurchaseLineLoc.SetFilter(Type, '%1|%2', PurchaseLineLoc.Type::"G/L Account",
                                                 PurchaseLineLoc.Type::Item);
        PurchaseLineLoc.SetFilter("Quantity Received", '<>%1', 0);
        if PurchaseLineLoc.FindSet(false, false) then begin
            repeat
                // PurchaseLineLoc.TestField("KVSM Pedimento No.");
                // PurchaseLineLoc.TestField("KVSM Pedimento Code");
                // PurchaseLineLoc.TestField("KVSM Pedimento Date");
            until PurchaseLineLoc.Next() = 0;
        end;
    end;

}